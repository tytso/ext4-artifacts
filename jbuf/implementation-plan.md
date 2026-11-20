# Design and Implementation Plan: Dedicated Block Cache (`jbuf`) for `ext4`, `jbd2`, and `ocfs2`

## 1. Goals and Summary

Replace the use of the deprecated generic Linux buffer cache (`fs/buffer.c` and
`include/linux/buffer_head.h`, `struct buffer_head`) in `ext4` and `jbd2`—and
across `ocfs2`'s metadata, journaling, and buffered file I/O paths—with a
dedicated block cache subsystem (`jbuf`, located in `fs/jbd2/jbuf.c` and
`include/linux/jbd2_jbuf.h`).

This achieves four concrete outcomes:

1. **`ext4` steps completely away from `CONFIG_BUFFER_HEAD`**: `ext4` and `jbd2`
   drop `select BUFFER_HEAD` in Commit 2 and operate cleanly on
   `CONFIG_BUFFER_HEAD=n` kernels. `ocfs2` stays fully functional as `jbd2`
   migrates off `buffer_head` (keeping its legacy `__blockdev_direct_IO()` path
   as-is in this series).
2. **Eliminate unused `fs/buffer.c` code and complexity**: Strip legacy APIs,
   `IOMAP_F_BUFFER_HEAD` plumbing, `get_block_t`-driven metadata writeback,
   highmem-only global accounting on 64-bit kernels, and dead state flags that
   neither `ext4`, `jbd2`, nor `ocfs2` need.
3. **Unify `struct journal_head` into `struct jbuf`**: Eliminate the separate
   `jbd2_journal_head_cache` slab, `bh->b_private`/`jh->b_bh` pointer chasing,
   `include/linux/journal-head.h`, the `PREEMPT_RT`-unfriendly
   `BH_JournalHead` `bit_spin_lock`, and throwaway shadow `buffer_head`
   allocations during journal commit, shrinking per-journaled-buffer slab
   footprint from ~256 B (across two 128 B slab objects) to ~192 B (one packed
   3-cacheline object).
4. **Detect cross-type metadata aliasing on corrupted images**: Add an atomic
   `enum jbuf_meta_type` field (`b_meta_type`) paired with `JBH_Verified` so
   any block verified or claimed for one role (e.g., a block allocation bitmap)
   cannot be silently trusted or reused as another role (e.g., an external
   xattr block, extent node, or directory block), returning `-EFSCORRUPTED` and
   eliminating a major class of `syzkaller` fuzzing failures.

Companion architecture documentation is in
`Documentation/filesystems/ext4/block_cache.rst` (to be committed at the end of
the series in Commit 20).

An **optional, approval-gated** design for keeping the kernel's superblock
coherent with userspace writes to the mounted block device (`e2fsck` of a
read-only mounted root, boot code in the first fs block, and legacy
`tune2fs`/`e2label` direct superblock writes) is in §7. Nothing in §7 is part
of the base series unless approved.

---

## 2. Baseline Codebase Audit (v7.4-rc2)

| Component | Baseline Size / Usage | Notes |
| --- | --- | --- |
| `fs/buffer.c` + `include/linux/buffer_head.h` | 3,006 + 594 = 3,600 LOC | Plus 148 LOC in `mm/migrate.c` (`__buffer_migrate_folio`) |
| `fs/jbd2/*.c` + `include/linux/{jbd2,journal-head}.h` | 9,604 + 1,955 = 11,559 LOC | Uses **36** `buffer_head.h` symbols; **only** callers in tree are `ext4` and `ocfs2` |
| `fs/ext4/` (production + 23 KUnit test files) | 662 `struct buffer_head` refs across 52 files | Uses **63** `buffer_head.h` symbols; **0** calls to `fs/mpage.c` (3 unused `#include`s), **0** calls to `fs/direct-io.c`, **0** `IOMAP_F_BUFFER_HEAD` uses |
| `fs/ocfs2/` | 838 `struct buffer_head` refs across 46 files | Uses **37** `buffer_head.h` symbols; calls `mpage_readahead()` and `mpage_writepages()` in `fs/ocfs2/aops.c`, plus `__blockdev_direct_IO()` (`CONFIG_LEGACY_DIRECT_IO`) in `ocfs2_direct_IO()` |
| External VFS helpers (`fs/mbcache.c`, `fs/quota/`, `fs/crypto/`, `fs/verity/`, `fs/dax.c`, `include/trace/events/{ext4,jbd2}.h`) | **0** `struct buffer_head` parameters | Clean boundary — none pass or accept `struct buffer_head *` |

---

## 3. Core Architectural Decisions & Hazards Addressed Upfront

### 3.1 Placement, Naming, and Build Model (`fs/jbd2/jbuf.c`, `CONFIG_JBUF`)

* **Files**: `fs/jbd2/jbuf.c`, `fs/jbd2/jbuf-test.c`, `fs/jbd2/.kunitconfig`,
  `include/linux/jbd2_jbuf.h`, `include/trace/events/jbuf.h`, and
  `Documentation/filesystems/ext4/block_cache.rst`. The header is named
  `jbd2_jbuf.h` (rather than `jbuf.h`) to make it clear that it is scoped to
  `jbd2` and its two users, `ext4` and `ocfs2`; `include/linux/jbd2.h`
  includes `<linux/jbd2_jbuf.h>` in place of `<linux/buffer_head.h>` (and, from
  Commit 11, in place of `<linux/journal-head.h>`).
* **Identifiers**: `struct jbuf`, `struct jbuf_cache`, `struct jbuf_mmb`,
  `enum jbuf_state_bits` (`JBH_*`), `enum jbuf_meta_type` (`JBUF_TYPE_*`),
  `jbuf_get_block_t`, and `jbuf_*` / `sb_jbuf_*` functions. Zero identifier
  collisions exist for `jbuf` or `JBH_` anywhere in the kernel tree.
* **Why `fs/jbd2/jbuf.c` works cleanly for `ext4` no-journal mode and modular
  builds**:
  * Both `EXT4_FS` and `OCFS2_FS` unconditionally `select JBD2` in Kconfig, so
    `ext4` mounted in no-journal mode (`-o noload` or `!has_journal`) always
    has `jbuf` available.
  * In `fs/jbd2/Kconfig` and `fs/jbd2/Makefile` (with `obj-y += jbd2/` in
    `fs/Makefile`, matching the `fs/quota/` pattern), `config JBUF` is a
    hidden `bool` selected by `config JBD2`. Just like `CONFIG_BUFFER_HEAD` and
    `CONFIG_FS_IOMAP`, `fs/jbd2/jbuf.c` calls core internal helpers that are
    not exported to loadable modules (`guard_bio_eod()`, `__folio_mark_dirty()`,
    `get_mem_cgroup_from_folio()`, `set_active_memcg()`, `mem_cgroup_put()`,
    `lru_cache_disabled()`, `cpu_is_isolated()`) and provides inline-stubbed
    hooks for `mm/filemap.c`, `mm/migrate.c`, `mm/folio.c`, and
    `mm/backing-dev.c`. Building `jbuf.o` into `vmlinux` when `CONFIG_JBUF=y`
    allows `jbd2.ko`, `ext4.ko`, `ocfs2.ko`, `jbuf-test.ko`, and
    `ext4-test.ko` to build cleanly as either built-in (`=y`) or loadable
    modules (`=m`). Because `struct journal_head` fields are embedded directly
    in `struct jbuf` without `fs/jbd2/jbuf.c` calling into `jbd2.ko`, there is
    zero circular dependency between `jbuf.o` and `jbd2.ko`.

### 3.2 Private Metadata `address_space` (`struct jbuf_cache`) & XFS-Style Loose Coherency

**Critical constraint**: `struct jbuf` objects **must never** be attached to
folios in `bdev->bd_mapping`. `bdev->bd_mapping->a_ops` is `def_blk_aops`
(`block/fops.c`), which unconditionally interprets `folio->private` as either
`struct buffer_head *` (`CONFIG_BUFFER_HEAD=y`) or `struct iomap_folio_state *`
(`CONFIG_BUFFER_HEAD=n`) during reclaim, writeback, migration, invalidation,
and userspace reads of `/dev/<bdev>`.

**Design**: Mirror `struct bdev_inode` / `blockdev_superblock` (`block/bdev.c`)
with a small internal pseudo-filesystem `jbuf_superblock` (`SB_I_CGROUPWB`) and
per-device cache handle:

```c
struct jbuf_cache {
	struct block_device	*jc_bdev;
	struct address_space	*jc_mapping;	/* &vfs_inode.i_data */
	struct inode		vfs_inode;
};
```

Key properties of this design:

1. **Mount ordering**: `ext4_fill_super()` calls `ext4_load_super()` (line
   5485) *before* setting `sb->s_op = &ext4_sops` (line 5593); calling
   `new_inode(sb)` there would allocate from generic `inode_cachep` and later
   free into `ext4_inode_cachep` on unmount. `jbuf_cache_create(bdev, size)`
   has zero dependency on filesystem `super_operations` and also works
   standalone in KUnit tests.
2. **Exact BDI writeback (including external journals)**: A 2-line check in
   `mm/backing-dev.c:inode_to_bdi()` (alongside `sb_is_blkdev_sb()`) returns
   `JBUF_CACHE_I(inode)->jc_bdev->bd_disk->bdi`, routing background flusher
   threads and cgroup writeback to the exact device for both `sb->s_bdev` and
   an external `s_journal_bdev_file`.
3. **Error tracking (`wb_err`)**: `ext4_check_bdev_write_error()` and
   `jbd2_check_fs_dev_write_error()` sample `jc_mapping->wb_err` so background
   checkpoint and `nojournal` metadata writeback `-EIO` errors are reliably
   caught.
4. **Sync and barrier ordering**: In `ext4_sync_fs()` (fixing `nojournal` mode,
   which previously issued `blkdev_issue_flush()` before VFS ran
   `sync_blockdev()`) and `ocfs2_sync_fs()`, `wait == 0` invokes
   `jbuf_cache_sync_nowait()` and `wait == 1` invokes `jbuf_cache_sync()`
   *before* `blkdev_issue_flush()`.
5. **Large Block Size (`FS_LBS`) and online resize**:
   `jbuf_cache_set_blocksize()` sets `i_blkbits` and
   `mapping_set_folio_min_order(jc_mapping, get_order(size))` (supporting 1 KiB
   up to 64 KiB blocks); device end-of-device checks consult
   `bdev_nr_bytes(jc->jc_bdev)` dynamically so online resize works seamlessly.
6. **XFS-equivalent loose coherency with `bdev->bd_mapping`**:
   Matching `fs/xfs/` (`xfs_alloc_buftarg()`, `xfs_close_devices()`, and
   `xfs_ioc_setlabel()`):
   * **While mounted**: Legacy userspace readers (notably bootloaders, `blkid`,
     and `dumpe2fs`) can freely open `/dev/<bdev>` read-only and read through
     `bdev->bd_mapping` (`def_blk_aops`, whether backed by `buffer_head` or
     `iomap`), completely undisturbed by `jbuf`. No coherency synchronization
     between `jc_mapping` and `bdev->bd_mapping` is needed during normal
     runtime, `sync_fs()`, or `freeze()`.
   * **Writes to a mounted block device are not supported by the base
     series.** *(Correction to the previous revision of this plan, which said
     they are blocked by default.)* The block layer still **permits** them by
     default: `sb_open_mode()` passes `BLK_OPEN_RESTRICT_WRITES`, but
     `block/bdev.c` enforces it only when `CONFIG_BLK_DEV_WRITE_MOUNTED=n`
     (the Kconfig default is `y`, and its help text cites fsck of a read-only
     mounted root and changing ext4 features while mounted as reasons) or
     when booted with `bdev_allow_write_mounted=0`. Such writes land only in
     `bdev->bd_mapping`. Today `ext4` sees them because `sbi->s_sbh` (and
     `sbi->s_es`, which points into it) lives in the same page cache; with
     `jbuf` the kernel does not see them and **silently overwrites** them the
     next time it writes the affected block (journal checkpoint or
     `ext4_commit_super()`). Known affected userspace writers:
     1. Legacy `tune2fs`/`e2label` direct superblock updates.
     2. `e2fsck` of a read-only mounted root followed by a read-write remount:
        `__ext4_remount()` re-reads `s_state` because "it may have been
        changed by e2fsck since we originally mounted the partition", and
        with a stale `s_es` the remount would write back the old `s_state`,
        error information, `s_mnt_count`, and `s_lastcheck`, forcing repeated
        full checks on later boots.
     3. Bootloader installers such as `extlinux --install`, which write the
        partition boot sector (within bytes 0–1023) while the file system is
        mounted; with a block size ≥ 2 KiB those bytes share fs block 0 with
        the superblock, so the kernel's next write of block 0 reverts them.

     Optional handling for all three is proposed in §7 (pending approval).
   * **At mount**: `jbuf_cache_create()` runs `sync_blockdev(bdev)` +
     `invalidate_bdev(bdev)` before reading any metadata so prior offline
     userspace buffered writes (`mke2fs`, `tune2fs`, `e2fsck`) are on disk.
   * **At unmount (and online label/UUID ioctls)**: `ext4_put_super()` and
     `ocfs2_dismount_volume()` release `s_sbh` / `osb_super_bh`, call
     `jbuf_cache_destroy()` (flushing `jc_mapping` and asserting
     `WARN_ON_ONCE(!mapping_empty(jc_mapping))`), and call
     `invalidate_bdev(bdev)` (plus on `FS_IOC_SETFSLABEL` /
     `EXT4_IOC_SETFSUUID` after updating superblocks, just like XFS) so
     post-unmount tools (`e2fsck`, `dumpe2fs`, `blkid`) do not see stale pages
     cached in `bdev->bd_mapping` while the filesystem was mounted.
7. **VM observability & `drop_caches`**: `jbuf_superblock` participates in
   `/proc/sys/vm/drop_caches` automatically, and `nr_jbuf_cache_pages()` adds
   to `Buffers:` in `/proc/meminfo` (alongside `nr_blockdev_pages()`).

### 3.3 Orphaned Truncated Folio Safety (`JBUF_FOLIO_TAG`)

In `ext4` `data=journal` mode, truncating a file whose folio has buffers pinned
by a committing transaction leaves an orphaned folio on the VM LRU with
`folio->mapping == NULL` and `folio->private != NULL`. Both
`mm/filemap.c:filemap_release_folio()` and `mm/migrate.c` (line 1330) fall back
to `try_to_free_buffers(folio)` when `folio->mapping == NULL`, which would free
a `struct jbuf` into `bh_cachep` on kernels with `CONFIG_BUFFER_HEAD=y`.

`jbuf` tags `folio->private` with bit 0 set (`JBUF_FOLIO_TAG = 1UL`). Only
three lines in `mm/` touch `folio_get_private()` directly (all in
`mm/migrate.c`, forwarding the opaque pointer or testing non-NULL), so the tag
is completely transparent to `mm/`. `filemap_release_folio()` and
`mm/migrate.c` check `folio_has_jbufs(folio)` and dispatch to
`try_to_free_jbufs(folio)`. As a bonus, `folio_jbufs(folio)` warns immediately
if ever handed an untagged `buffer_head` or `iomap` folio.

### 3.4 Core Identifier Rename Mapping

| Legacy `buffer_head.h` | New `jbd2_jbuf.h` |
| --- | --- |
| `struct buffer_head` | `struct jbuf` |
| *(implicit `bdev->bd_mapping`)* | `struct jbuf_cache` |
| `struct mapping_metadata_bhs` / `mmb_*` | `struct jbuf_mmb` / `jbuf_mmb_*` |
| `get_block_t` | `jbuf_get_block_t` |
| `enum bh_state_bits` / `BH_*` | `enum jbuf_state_bits` / `JBH_*` |
| `BUFFER_FNS` / `TAS_BUFFER_FNS` | `JBUF_FNS` / `TAS_JBUF_FNS` |
| `buffer_<flag>` / `{set,clear}_buffer_<flag>` | `jbuf_<flag>` / `{set,clear}_jbuf_<flag>` |
| `folio_buffers` / `bh_offset` / `folio_set_bh` | `folio_jbufs` / `jbuf_offset` / `folio_set_jbuf` |
| `kmap_local_bh` / `kunmap_local_bh` | `kmap_local_jbuf` / `kunmap_local_jbuf` |
| `alloc_buffer_head` / `free_buffer_head` | `jbuf_alloc` / `jbuf_free` |
| `folio_alloc_buffers` / `create_empty_buffers` | `folio_alloc_jbufs` / `jbuf_create_empty` |
| `get_bh` / `put_bh` / `brelse` / `__brelse` | `jbuf_get` / `jbuf_put` / `jbuf_relse` / `__jbuf_relse` |
| `bforget` / `__bforget` | `jbuf_forget` / `__jbuf_forget` |
| `{lock,trylock,unlock}_buffer` / `wait_on_buffer` | `jbuf_{lock,trylock,unlock}` / `jbuf_wait` |
| `__find_get_block{,_nonatomic}` | `jbuf_find_get_block{,_nonatomic}` (takes `struct jbuf_cache *`) |
| `bdev_getblk` / `__getblk` / `getblk_unmovable` | `jbuf_getblk_gfp` / `jbuf_getblk` / `jbuf_getblk_unmovable` |
| `__bread` / `__bread_gfp` / `sb_bread` / `sb_getblk{,_gfp}` | `jbuf_bread` / `jbuf_bread_gfp` (plus `ext4_sb_*` / `ocfs2` helpers) |
| `bh_read{,_nowait}` / `bh_readahead_batch` / `bh_uptodate_or_lock` | `jbuf_read{,_nowait}` / `jbuf_readahead_batch` / `jbuf_uptodate_or_lock` |
| `bh_submit` / `bh_end_{read,write,async_write}` / `bio_endio_bh` | `jbuf_submit` / `jbuf_end_{read,write,async_write}` / `bio_endio_jbuf` |
| `mark_buffer_dirty` / `{write,sync,__sync}_dirty_buffer` | `jbuf_mark_dirty` / `jbuf_{write,sync,__sync}_dirty` |
| `mark_buffer_write_io_error` | `jbuf_mark_write_io_error` |
| `clean_bdev_aliases` / `clean_bdev_bh_alias` | `jbuf_clean_aliases` / `jbuf_clean_alias` |
| `map_bh` | `jbuf_map` |
| `try_to_free_buffers` / `invalidate_bh_lrus{,_cpu}` | `try_to_free_jbufs` / `jbuf_invalidate_lrus{,_cpu}` |
| `block_{dirty,invalidate,read_full,write_full}_folio` | `jbuf_{dirty,invalidate,read_full,write_full}_folio` |
| `__block_write_begin` / `block_{write_end,commit_write}` | `__jbuf_write_begin` / `jbuf_{write_end,commit_write}` |
| `folio_zero_new_buffers` / `block_is_partially_uptodate` | `jbuf_zero_new` / `jbuf_is_partially_uptodate` |
| `block_page_mkwrite` / `buffer_migrate_folio{,_norefs}` | `jbuf_page_mkwrite` / `jbuf_migrate_folio{,_norefs}` |

Existing local pointer variable names (`struct jbuf *bh`) are intentionally
kept at call sites in `ext4`, `jbd2`, and `ocfs2` to minimize cosmetic diff
churn; a global variable rename via open-source Coccinelle can be done later as
a separate cleanup after dead-code elimination.

### 3.5 `ocfs2` Scope (`mpage` Buffered I/O vs. Legacy Direct I/O)

* **Buffered `mpage` (`ocfs2_readahead`, `ocfs2_writepages`)**: `ocfs2`'s
  regular-file `address_space_operations` mixes `block_read_full_folio()`,
  `__block_write_begin()`, `block_dirty_folio()`, and `mpage_{readahead,writepages}()`.
  Because a single folio cannot mix `buffer_head` and `jbuf`, Commit 1 includes
  `jbuf_mpage_readahead()` and `jbuf_mpage_writepages()` in `fs/jbd2/jbuf.c` so
  all `ocfs2` page-cache folios use `struct jbuf` consistently.
* **Legacy Direct I/O (`ocfs2_direct_IO`)**: Kept on `__blockdev_direct_IO()`
  (`CONFIG_LEGACY_DIRECT_IO` / `CONFIG_BUFFER_HEAD`) in this series.
  `fs/direct-io.c` passes a stack-allocated `struct buffer_head map_bh` to
  `ocfs2_dio_get_block()` solely as a block-mapping output struct (never cached,
  never attached to a folio, never passed to JBD2), so `ocfs2_dio_get_block()`
  keeps its `struct buffer_head *bh_result` signature while converting its
  internal inode metadata buffer (`di_bh`) to `struct jbuf *`.

---

## 4. Commit-by-Commit Implementation Plan

Every commit must independently:

1. Compile cleanly with **zero warnings** (`CONFIG_EXT4_FS=y`,
   `CONFIG_OCFS2_FS=m`, `CONFIG_EXT4_KUNIT_TESTS=y/m`,
   `CONFIG_JBUF_KUNIT_TEST=y/m`, plus a `CONFIG_BUFFER_HEAD=n` build for `ext4`
   from Commit 2 onward).
2. Pass `./scripts/checkpatch.pl -g HEAD`.
3. Pass all KUnit suites (`jbuf` and `ext4`, including isolated single-test
   runs and modular `=m` link verification).
4. Pass `kvm-xfstests smoke` on a `--lockdep --kasan --extra-debug` kernel
   without regressions, KASAN splats, lockdep warnings, or unmount cache-leak
   warnings.
5. Omit `Signed-off-by:` tags per `Documentation/process/coding-assistants.rst`
   (leaving DCO sign-off to the human maintainer) and include transparent tool
   and test notes per `Documentation/process/generated-content.rst`.

---

### Phase 1: Bootstrap Abstraction & Switch Filesystems

#### Commit 1: `jbd2: introduce jbuf block cache abstraction and KUnit tests`

* **Files added**:
  * `include/linux/jbd2_jbuf.h` (cloned from `include/linux/buffer_head.h`,
    renamed to `jbuf`/`JBH_*`, plus `struct jbuf_cache` lifecycle APIs,
    `struct jbuf_mmb`, `JBUF_FOLIO_TAG` helpers, and `!CONFIG_JBUF` stubs).
  * `fs/jbd2/jbuf.c` (cloned from `fs/buffer.c`,
    `mm/migrate.c:__buffer_migrate_folio`, and
    `fs/mpage.c:mpage_{readahead,writepages}`, adapted to `struct jbuf_cache` +
    `jbuf_superblock` pseudo-fs + `jbuf_meta_aops` + `nr_jbuf_cache_pages()`).
  * `fs/jbd2/jbuf-test.c` (comprehensive KUnit suite `jbuf` following
    `writing-kunit-tests.md`).
  * `fs/jbd2/.kunitconfig` (standalone UML KUnit config enabling `CONFIG_KUNIT=y`,
    `CONFIG_JBD2=y`, `CONFIG_JBUF_KUNIT_TEST=y`).
* **Minimal core plumbing**:
  * `fs/Makefile`, `fs/jbd2/Kconfig`, `fs/jbd2/Makefile`: Descend into
    `fs/jbd2/` via `obj-y += jbd2/`; add `config JBUF` (`bool`) and
    `config JBUF_KUNIT_TEST` (`tristate`, `depends on JBUF && KUNIT`,
    `default KUNIT_ALL_TESTS`).
  * `include/linux/cpuhotplug.h`: Add `CPUHP_FS_JBUF_DEAD` alongside
    `CPUHP_FS_BUFF_DEAD`.
  * `include/uapi/linux/magic.h` (or internal magic in `fs/jbd2/jbuf.c`):
    `JBUF_SUPER_MAGIC` for `jbuf_superblock` and `sb_is_jbuf_sb()`.
  * `mm/mm_init.c`: Call `jbuf_init()` next to `buffer_init()`.
  * `mm/backing-dev.c`: Route `sb_is_jbuf_sb(sb)` in `inode_to_bdi()` to
    `jc->jc_bdev->bd_disk->bdi`.
  * `mm/filemap.c`, `mm/migrate.c`: Dispatch `folio_has_jbufs(folio)` to
    `try_to_free_jbufs(folio)` for orphaned folios (`mapping == NULL`).
  * `mm/folio.c`: Call `jbuf_invalidate_lrus_cpu()` in
    `lru_add_and_bh_lrus_drain()`.
  * `mm/vmscan.c`: Include `jbuf_heads_over_limit` alongside
    `buffer_heads_over_limit`.
  * `block/bdev.c` (`nr_blockdev_pages()`): Include `nr_jbuf_cache_pages()` so
    `/proc/meminfo` `Buffers:` reflects `jbuf_cache` pages.
* **KUnit test coverage in `fs/jbd2/jbuf-test.c`**:
  * Alloc/free lifecycle and initial struct invariants.
  * State-bit helpers, atomic test-and-set/clear, and `JBH_Uptodate` ordering.
  * Buffer locking (`jbuf_lock`, `jbuf_trylock`, `jbuf_unlock`, `jbuf_wait`).
  * Reference counting (`jbuf_get`, `jbuf_put`, `jbuf_relse`, `jbuf_forget`
    clearing dirty/EIO/mmb state, underflow warning check via
    `kunit_warning_suppress`).
  * Multi-block folio ring creation, offset calculation, `kmap_local_jbuf`,
    `folio_has_jbufs` tag verification, and `try_to_free_jbufs` across 1 KiB,
    2 KiB, and 4 KiB block sizes (busy due to `b_count`, `JBH_Dirty`,
    `JBH_Lock`, writeback; stale folio dirty cancellation when all buffers
    clean; orphaned `folio->mapping == NULL` release via
    `filemap_release_folio()`).
  * Folio helper behaviors: `jbuf_dirty_folio`, `jbuf_invalidate_folio`
    (partial vs. full folio), `jbuf_is_partially_uptodate` (boundary matrix via
    `KUNIT_CASE_PARAM`), `jbuf_commit_write`, `jbuf_zero_new`, and
    `jbuf_check_dirty_writeback`.
  * `struct jbuf_mmb` operations: init, mark dirty, duplicate mark idempotency,
    multiply-claimed block protection across two `jbuf_mmb` lists, invalidate,
    and automatic detach on `jbuf_forget` and `try_to_free_jbufs`.
  * `struct jbuf_cache` operations (backed by an in-memory fake/test
    `block_device` / mapping): `jbuf_getblk`, atomic vs. sleepable
    `jbuf_find_get_block`, per-CPU LRU hit/promotion/16-entry eviction and
    `jbuf_invalidate_lrus`, blocksize reconfiguration, `jbuf_clean_aliases`
    range precision (exact boundary blocks untouched), and
    `jbuf_migrate_folio` / `jbuf_migrate_folio_norefs` (succeeds after LRU
    drain, returns `-EAGAIN` when caller holds `b_count` or `JBH_Lock`).
* **Risk**: Low — no existing filesystem selects `CONFIG_JBUF` yet.

#### Commit 2: `jbd2, ext4, ocfs2: convert from buffer_head to jbuf block cache`

*(Note: `jbd2`, `ext4`, and `ocfs2` switch together in one commit to preserve
`git bisect` across all configs, because `include/linux/jbd2.h` buffer APIs are
called directly by both `ext4` and `ocfs2`.)*

* **Kconfig updates**:
  * `fs/jbd2/Kconfig`: Add `select JBUF` under `config JBD2`.
  * `fs/ext4/Kconfig`: **Remove `select BUFFER_HEAD`** from `config EXT4_FS`.
  * `fs/ext4/.kunitconfig`: Add `CONFIG_JBUF_KUNIT_TEST=y` so KUnit runs
    exercise both `jbuf` and all 23 `ext4` suites together.
* **`jbd2` changes** (`include/linux/jbd2.h`, `include/linux/journal-head.h`,
  `fs/jbd2/{journal,transaction,commit,checkpoint,recovery,revoke}.c`):
  * Replace `struct buffer_head` and `buffer_*`/`bh_*` calls with `struct jbuf`
    and `jbuf_*` equivalents (keeping `struct jbuf *bh` variable names);
    replace `BH_PrivateStart` with `JBH_PrivateStart`; switch
    `include/linux/jbd2.h` from `#include <linux/buffer_head.h>` to
    `#include <linux/jbd2_jbuf.h>`.
  * Extend `journal_t` with `struct jbuf_cache *j_dev_cache` and
    `struct jbuf_cache *j_fs_cache`, passed into `jbd2_journal_init_dev()` and
    `jbd2_journal_init_inode()`.
  * Route `jbd2_journal_get_descriptor_buffer()`, `jbd2_fc_get_buf()`,
    `journal_load_superblock()`, `recovery.c` (`jread`, `PASS_REPLAY`, and
    post-recovery sync), `revoke.c` (`jbuf_find_get_block_nonatomic`),
    `jbd2_journal_clear_features()` / discard range truncation, and
    `jbd2_{init,check}_fs_dev_write_error()` (`jc_mapping->wb_err`) through
    `j_dev_cache` and `j_fs_cache`.
* **`ext4` changes** (`fs/ext4/*.c`, `fs/ext4/*.h`, and 22 `fs/ext4/*-test.c`
  files):
  * Add `struct jbuf_cache *s_bcache` and `struct jbuf_cache *s_journal_bcache`
    to `struct ext4_sb_info`.
  * Create `s_bcache` in `ext4_load_super()`, update blocksize via
    `jbuf_cache_set_blocksize()` alongside `sb_set_blocksize()`, sync in
    `ext4_sync_fs()` (`nowait` when `!wait`, `sync` before flush when `wait` in
    `nojournal` mode) and R/O `__ext4_remount()`, release `sbi->s_sbh` *before*
    calling `jbuf_cache_destroy(sbi->s_bcache)` + `invalidate_bdev(sb->s_bdev)`
    in `ext4_put_super()` and `ext4_fill_super()` error paths, and call
    `invalidate_bdev(sb->s_bdev)` after online superblock label/UUID updates
    (`FS_IOC_SETFSLABEL` / `EXT4_IOC_SETFSUUID`, matching XFS).
  * Create/destroy `s_journal_bcache` in `ext4_get_journal_blkdev()` /
    `ext4_open_dev_journal()` / `ext4_put_super()`.
  * Update `ext4_sb_getblk*()`, `ext4_sb_find_get_block*()`,
    `ext4_sb_bread*()`, `ext4_readdir()` metadata readahead, and
    `ext4_check_bdev_write_error()` to use `sbi->s_bcache`.
  * Convert all remaining `buffer_head` structs, state macros (`EXT4_MAP_*`,
    `JBH_BITMAP_UPTODATE`), `struct jbuf_mmb` (`i_metadata_bhs`), data-path
    `ext4_{aops,da_aops,journalled_aops}`, `readpage.c`, `page-io.c`, drop
    unused `<linux/mpage.h>` / `<linux/buffer_head.h>` includes, and update
    mock `jbuf` fixtures across `fs/ext4/*-test.c`.
* **`ocfs2` changes** (`fs/ocfs2/*.{c,h}`):
  * Add `struct jbuf_cache *s_bcache` to `struct ocfs2_super`; initialize in
    `ocfs2_initialize_super()` after `sb_set_blocksize()`, flush in
    `ocfs2_sync_blockdev()` / `ocfs2_sync_fs()`, and release `osb_super_bh`
    before `jbuf_cache_destroy()` + `invalidate_bdev(sb->s_bdev)` in
    `ocfs2_dismount_volume()` and mount failure cleanup.
  * Convert all metadata, journaling, `buffer_head_io.{c,h}`, `uptodate.{c,h}`,
    quota, refcount, xattr, dir, and `ocfs2_aops` buffered I/O (including
    `jbuf_mpage_{readahead,writepages}`) to `struct jbuf`.
  * Keep `ocfs2_dio_get_block()` in `fs/ocfs2/aops.c` on legacy
    `__blockdev_direct_IO()` (`struct buffer_head *bh_result`).
* **Extra verification on Commit 2**:
  * Build and run `kvm-xfstests smoke` on **both** standard config and a
    `CONFIG_BUFFER_HEAD=n` ext4 kernel config!
  * Run extra `kvm-xfstests` spot-checks covering `fast_commit`, `inline`,
    `inline_1k`, `quota`, external journal device, online resize (`ext4/056`,
    `resize` group), `-o buffered_iomap`, and userspace read access to a
    mounted block device (`dumpe2fs` / `dd if=/dev/vdc`).

---

### Phase 2: Incremental Dead-Code Elimination & Simplifications

Each commit removes one self-contained unused feature from `fs/jbd2/jbuf.c` /
`include/linux/jbd2_jbuf.h` and is independently built and verified with KUnit and
`kvm-xfstests smoke`.

#### Commit 3: `jbd2: jbuf: remove unused cont_write_begin and generic_cont_expand_simple`

* Delete `cont_write_begin()`, `cont_expand_zero()`, and
  `generic_cont_expand_simple()` (~125 LOC), which exist only for filesystems
  that do not support sparse holes (e.g., FAT, affs, hpfs).

#### Commit 4: `jbd2: jbuf: remove unused jbuf_truncate_page and generic_jbuf_bmap`

* Delete `jbuf_truncate_page()` and `generic_jbuf_bmap()` (~85 LOC). Both
  `ext4` (`ext4_block_truncate_page()`, `iomap_bmap()`) and `ocfs2`
  (`ocfs2_zero_tail()`, `ocfs2_extent_map_get_blocks()`) provide their own
  implementations.

#### Commit 5: `jbd2: jbuf: remove unused jbuf_write_begin, generic_jbuf_write_end, and iomap_to_jbuf`

* Delete `jbuf_write_begin()`, `generic_jbuf_write_end()`, and
  `iomap_to_jbuf()` (~145 LOC), folding `__jbuf_write_begin_int()` directly
  into `__jbuf_write_begin()` and removing `#include <linux/iomap.h>` from
  `fs/jbd2/jbuf.c` (neither `ext4` nor `ocfs2` uses `IOMAP_F_BUFFER_HEAD`).

#### Commit 6: `jbd2: jbuf: remove unused readahead, boundary, and state-bit helpers`

* Delete `write_boundary_block()`, `jbuf_readahead()`, `jbuf_read_batch()`,
  `__jbuf_readahead()`, `sb_jbuf_readahead()`, `sb_jbuf_read_unmovable()`,
  `get_nth_jbuf()`, `jbuf_waitq_head()`, `end_jbuf_read_sync()` export, and
  unused `JBH_Defer_Completion` (plus the corresponding boundary-block hook in
  `jbuf_mpage_writepages()`).

#### Commit 7: `jbd2: jbuf: replace generic write_full_folio with streamlined metadata writeback`

* Neither `ext4` (uses `ext4_writepages()` / `ext4_bio_write_folio()`) nor
  `ocfs2` (uses `jbuf_mpage_writepages()`) calls `jbuf_write_full_folio()` on
  file data.
* Delete `jbuf_write_full_folio()` and the 160-line `__jbuf_write_full_folio()`
  (hole/delalloc mapping via `get_block_t`, EOF-straddle mmap zeroing, and
  `ENOSPC` recovery path) and replace them with a ~45-line
  `jbuf_meta_write_folio()` specific to `jbuf_cache` metadata folios (whose
  buffers are already 1:1 physically mapped, never delayed/unwritten, and never
  fscrypt-encrypted).

#### Commit 8: `jbd2: jbuf: add dedicated jbuf tracepoints and drop block.h trace coupling`

* Create `include/trace/events/jbuf.h` with `jbuf_touch`, `jbuf_dirty`, and
  `jbuf_submit` tracepoints, replacing `<trace/events/block.h>` calls in
  `fs/jbd2/jbuf.c` (which were gated on `#ifdef CONFIG_BUFFER_HEAD` and
  silently compiled out on `!CONFIG_BUFFER_HEAD` builds).

#### Commit 9: `jbd2: jbuf: restrict jbuf_heads_over_limit accounting to CONFIG_HIGHMEM`

* Gate per-CPU `jbuf_accounting`, `recalc_jbuf_state()`, and
  `jbuf_heads_over_limit` behind `#ifdef CONFIG_HIGHMEM`. On 64-bit kernels
  (`!CONFIG_HIGHMEM`), `jbuf_heads_over_limit` compiles to constant `false`,
  eliminating `preempt_disable()` + per-CPU counter updates + periodic
  all-online-CPU walks from every `jbuf_alloc()` and `jbuf_free()`, while
  normal slab/LRU reclaim continues to reclaim clean folios and `jbuf`s.

---

### Phase 3: JBD2 `journal_head` Unification & Layout/I/O Optimizations

Decomposed into five small, reviewable, bisect-safe commits so each mechanical
vs. concurrency vs. I/O change is isolated.

#### Commit 10: `jbd2: embed struct journal_head inside struct jbuf`

* Move `struct journal_head b_jh` directly inside `struct jbuf` (removing
  `jh->b_bh` and converting `bh->b_private` to `struct jbuf *b_orig` used only
  by temporary shadow write buffers).
* Convert `bh2jh(bh)` to `&bh->b_jh` and `jh2bh(jh)` to
  `container_of(jh, struct jbuf, b_jh)`.
* Initialize `b_jh.b_state_lock` once in `jbuf_alloc()`.
* Delete `jbd2_journal_head_cache`, `jbd2_journal_init_journal_head_cache()`,
  `jbd2_journal_destroy_journal_head_cache()`, `journal_alloc_journal_head()`
  (and its `GFP_NOFS | __GFP_NOFAIL` OOM loop), `journal_free_journal_head()`,
  and `nr_journal_heads` slab tracking; simplify `jbd2_journal_add_journal_head()`
  to zero allocation and zero retry loop.
* **KUnit**: Verify embedded `b_jh` initialization, zero-alloc attach/detach,
  and cleanup assertions.

#### Commit 11: `jbd2: flatten journal_head into struct jbuf and remove journal-head.h`

* Hoist all `struct journal_head` members (`b_state_lock`, `b_jcount`,
  `b_jlist`, `b_modified`, `b_frozen_data`, `b_committed_data`,
  `b_transaction`, `b_next_transaction`, `b_tnext`, `b_tprev`,
  `b_cp_transaction`, `b_cpnext`, `b_cpprev`, `b_triggers`,
  `b_frozen_triggers`) directly into `struct jbuf`, and fold JBD2 state bits
  (`JBH_JBD`..`JBH_Verified`, `JBH_FSPrivateStart`) into `enum jbuf_state_bits`
  in `include/linux/jbd2_jbuf.h`.
* Convert `transaction_t` buffer/checkpoint list pointers and internal
  `fs/jbd2/*.c` helpers (`__jbd2_journal_file_buffer()`,
  `jbd2_journal_put_journal_head()` -> `jbd2_journal_put_jbuf()`, etc.) to
  operate directly on `struct jbuf *`, removing `bh2jh()`, `jh2bh()`,
  `J_ASSERT_JH`, and `include/linux/journal-head.h` (plus updating
  `MAINTAINERS` and `Documentation/filesystems/journalling.rst`).

#### Commit 12: `jbd2: replace JBH_JournalHead bit-spinlock and JBH_JBD with atomic b_jcount`

* Change `int b_jcount` to `atomic_t b_jcount`.
* Remove `JBH_JournalHead`, `jbd_lock_bh_journal_head()`,
  `jbd_unlock_bh_journal_head()`, and `JBH_JBD`:
  * `jbuf_jbd(bh)` becomes `atomic_read(&bh->b_jcount) > 0` (preserving exact
    semantics for all 19 `ocfs2` callers, `ext4`, and `jbd2`).
  * `jbd2_journal_add_jbuf(bh)`: `if (atomic_fetch_inc(&bh->b_jcount) == 0)
    jbuf_get(bh);`
  * `jbd2_journal_grab_jbuf(bh)`: `atomic_inc_not_zero(&bh->b_jcount)`.
  * `jbd2_journal_put_jbuf(bh)`: on `atomic_dec_and_test(&bh->b_jcount)`,
    calls `__jbuf_relse(bh)` (exact bijection between `0->1` and `1->0`
    transitions, safe even under concurrent `add_jbuf()` by a thread holding
    its own `b_count` reference).
* Audit all `b_frozen_data` and `b_committed_data` teardown paths to ensure
  they are always freed under `b_state_lock` before dropping the transaction's
  `b_jcount` reference; assert `b_jcount == 0`, `!b_frozen_data`, and
  `!b_committed_data` in `jbuf_free()`, and include `atomic_read(&bh->b_jcount)`
  in `jbuf_busy()`.
* **KUnit**: Add focused tests in `fs/jbd2/jbuf-test.c` covering
  `0->1->2->1->0->1` transitions, `grab_jbuf` on zero vs. positive `b_jcount`,
  and `try_to_free_jbufs()` rejection while `b_jcount > 0`.

#### Commit 13: `jbd2: submit journal shadow log writes directly via struct bio`

* In `fs/jbd2/journal.c:jbd2_journal_write_metadata_buffer()` and
  `fs/jbd2/commit.c` (`journal->j_wbuf[]`), replace allocating and freeing a
  temporary uncached `struct jbuf` for every logged metadata block with direct
  `struct bio` allocation/submission (`bio_add_folio_nofail()` for unescaped
  in-place buffers, `bio_add_virt_nofail()` for `b_frozen_data` slab copies,
  and `bio->bi_private = orig_bh` handled in `journal_end_shadow_bio()`).
* Delete `bh->b_orig` from `struct jbuf` and simplify `jbuf` I/O submission
  helpers now that no uncached `b_folio == NULL` shadow `jbuf`s are ever
  submitted through `jbuf_submit()`.

#### Commit 14: `jbd2: jbuf: pack struct jbuf layout for cacheline efficiency`

* Narrow `size_t b_size` to `u32` (max supported block size is 64 KiB) and
  `unsigned int b_jlist` / `b_modified` to discrete `u8` fields (never C
  bitfields, preventing tearing).
* Arrange `struct jbuf` into three 64-byte cachelines (cacheline 0: `b_state`,
  `b_this_page`, `b_folio`, `b_blocknr`, `b_data`, `b_bdev`, `b_size`,
  4-byte slot for `b_meta_type`, `b_count`, `b_uptodate_lock` = exact 64 B;
  cacheline 1: `b_jcount`, `b_state_lock`, `b_jlist`, `b_modified`,
  `b_transaction`, `b_next_transaction`, `b_tnext`, `b_tprev`,
  `b_cp_transaction`, `b_cpnext` = exact 64 B; cacheline 2: `b_cpprev`,
  `b_frozen_data`, `b_committed_data`, `b_triggers`, `b_frozen_triggers`,
  `b_assoc_buffers`, `b_mmb` = exact 64 B; total 192 B).
* Verify field offsets and holes with `pahole` and record exact before/after
  slab object sizes in the commit log.

---

### Phase 4: Metadata Block Type Identifier, Fuzzing Hardening & Docs

#### Commit 15: `jbd2: jbuf: add metadata block type tagging and conflict detection`

* Add `enum jbuf_meta_type`, `u32 b_meta_type` (occupying the reserved 4-byte
  slot in cacheline 0 at **zero** increase to `sizeof(struct jbuf)`),
  `jbuf_meta_type_name()`, and `trace_jbuf_type_mismatch` in
  `include/trace/events/jbuf.h`.
* Implement lockless, acquire/release-ordered type helpers:
  * `jbuf_get_type(bh)`
  * `jbuf_check_verified(bh, type)` -> `1` (verified as `type`), `0` (needs
    verification; does **not** mutate `b_meta_type` so failed speculative
    validation never poisons a valid block), or `-EFSCORRUPTED` (type conflict)
  * `jbuf_claim_type(bh, type)` -> `0` or `-EFSCORRUPTED` via
    `try_cmpxchg_acquire()`
  * `jbuf_set_verified(bh, type)` -> `0` or `-EFSCORRUPTED` (sets
    `JBH_Verified` only after `b_meta_type == type` is established)
  * `jbuf_change_type(bh, old_type, new_type)` -> `0` or `-EFSCORRUPTED`
    (clears `JBH_Verified`)
  * `jbuf_clear_verified(bh)` (clears `JBH_Verified`, preserves `b_meta_type`)
  * `jbuf_reset_type(bh)` (clears `JBH_Verified` and resets `b_meta_type` to
    `JBUF_TYPE_NONE` with release ordering)
* Wire `jbuf_reset_type()` into `__jbuf_forget()`, `discard_jbuf()`,
  `jbuf_clean_aliases()`, `jbd2_journal_forget()`, and
  `jbd2_journal_revoke()`.
* **KUnit tests in `fs/jbd2/jbuf-test.c`**:
  * Fresh buffer has `JBUF_TYPE_NONE` and `jbuf_check_verified()` returns `0`
    without modifying `b_meta_type`.
  * `jbuf_set_verified(bh, BLOCK_BITMAP)` followed by
    `jbuf_check_verified(bh, BLOCK_BITMAP)` returns `1`.
  * Cross-type conflict: `BLOCK_BITMAP` followed by `jbuf_check_verified()`,
    `jbuf_claim_type()`, or `jbuf_set_verified()` with `XATTR` returns
    `-EFSCORRUPTED` and leaves `b_meta_type == BLOCK_BITMAP` and
    `JBH_Verified == 1` intact (plus parameterized matrix across representative
    type pairs).
  * `jbuf_claim_type()` without `JBH_Verified` (non-`metadata_csum` flow)
    still rejects any different type with `-EFSCORRUPTED`.
  * `jbuf_clear_verified()` forces re-verification (`check_verified == 0`) for
    the same type while still rejecting a different type with `-EFSCORRUPTED`.
  * `jbuf_change_type(bh, DIR_DATA, DX_ROOT)` succeeds and clears
    `JBH_Verified`; calling with mismatched `old_type` fails with
    `-EFSCORRUPTED`.
  * Lifecycle reset via `jbuf_forget()`, `jbuf_invalidate_folio()`, and
    `jbuf_clean_aliases()` resets `b_meta_type` to `JBUF_TYPE_NONE` and permits
    claiming a new type `T2`.
  * Non-zero distinct type names for all enumerators in
    `[JBUF_TYPE_NONE, JBUF_TYPE_MAX)`.

#### Commit 16: `jbd2: tag journal superblock, log, and fast-commit buffers`

* Claim `JBUF_TYPE_JBD2_SUPER` on `journal->j_sb_buffer`,
  `JBUF_TYPE_JBD2_LOG` on cyclic journal ring blocks (descriptor, commit,
  revoke, and recovery `jread()` buffers — single type handles journal ring
  wrap-around without false positives while catching any alias between an
  in-filesystem journal inode and filesystem metadata), and
  `JBUF_TYPE_JBD2_FC` on fast-commit region buffers (`jbd2_fc_get_buf()` and
  fast-commit recovery reads).
* Leave target filesystem buffers written during `PASS_REPLAY` in `j_fs_cache`
  as `JBUF_TYPE_NONE` (`JBH_Verified == 0`) so the owning filesystem validates
  and binds their true metadata type on first access.

#### Commit 17: `ext4: enforce metadata block types on static global metadata`

* Wire `jbuf_check_verified()` / `jbuf_set_verified()` / `jbuf_claim_type()`
  into:
  * Primary & backup superblocks (`JBUF_TYPE_EXT4_SUPER`,
    `JBUF_TYPE_EXT4_BACKUP_SB`) in `super.c`, `resize.c`, `ioctl.c`.
  * Primary, backup, and reserved GDT blocks (`JBUF_TYPE_EXT4_GDT`,
    `JBUF_TYPE_EXT4_BACKUP_GDT`) in `super.c` and `resize.c`, using
    `jbuf_change_type(gdb, BACKUP_GDT, GDT)` when online resize activates a
    reserved GDT block.
  * Block allocation bitmaps (`JBUF_TYPE_EXT4_BLOCK_BITMAP`) in `balloc.c`,
    `mballoc.c`, and `resize.c`.
  * Inode allocation bitmaps (`JBUF_TYPE_EXT4_INODE_BITMAP`) in `ialloc.c` and
    `resize.c`.
  * Inode table blocks (`JBUF_TYPE_EXT4_INODE_TABLE`) in `inode.c`
    (`__ext4_get_inode_loc`), `ialloc.c`, and `resize.c`.
  * Multi-mount protection block (`JBUF_TYPE_EXT4_MMP`) in `mmp.c`.
  * Orphan file blocks (`JBUF_TYPE_EXT4_ORPHAN`) in `orphan.c`.
* Report type mismatches via `ext4_error*()` including block number,
  `jbuf_meta_type_name(jbuf_get_type(bh))`, and `jbuf_meta_type_name(expected)`.
* **KUnit tests**: Add targeted test cases in `balloc-test.c`, `ialloc-test.c`,
  `mmp-test.c`, `orphan-test.c`, and `resize-test.c` verifying cross-type
  rejection (e.g., block bitmap colliding with inode bitmap or inode table,
  valid `BACKUP_GDT -> GDT` online resize promotion).

#### Commit 18: `ext4: enforce metadata block types on dynamic metadata and data blocks`

* Replace remaining untyped `jbuf_verified()` / `set_jbuf_verified()` calls
  and claim types across dynamic metadata and journaled data:
  * Extent tree blocks (`JBUF_TYPE_EXT4_EXTENT`) in `extents.c` and `migrate.c`.
  * Indirect/dind/tind blocks (`JBUF_TYPE_EXT4_INDIRECT`) in `indirect.c`,
    `migrate.c`, and `resize.c` (`resize_inode` `dind` block).
  * Directory blocks (`JBUF_TYPE_EXT4_DIR_DATA`, `JBUF_TYPE_EXT4_DX_ROOT`,
    `JBUF_TYPE_EXT4_DX_NODE`) in `namei.c`, `dir.c`, and `inline.c`:
    * Explicit `jbuf_change_type(bh, EXT4_DIR_DATA, EXT4_DX_ROOT)` in
      `make_indexed_dir()`.
    * Careful handling of `EITHER` and linear-scan fallback over
      `is_dx_internal_node()` blocks (accepting `DX_ROOT` at lblk 0 and
      `DX_NODE` at lblk > 0 while rejecting all non-directory types and
      rejecting `DX_ROOT` as `DX_NODE` self-loops).
  * External xattr blocks (`JBUF_TYPE_EXT4_XATTR`) and `ea_inode` blocks
    (`JBUF_TYPE_EXT4_EA_INODE`) in `xattr.c`.
  * Out-of-line symlink blocks (`JBUF_TYPE_EXT4_SYMLINK`) in `symlink.c` and
    `namei.c`.
  * Quota blocks (`JBUF_TYPE_EXT4_QUOTA`) in `super.c`
    (`ext4_quota_{read,write}`).
  * Buffered / `data=journal` regular-file data blocks (`JBUF_TYPE_DATA`) in
    `inode.c` (catching data extents overlapping live cached metadata blocks).
  * Remove legacy untyped `set_jbuf_verified()` helper once all callers pass an
    explicit `enum jbuf_meta_type`.
* **KUnit tests**: Add tests in `xattr-test.c`, `extents-test.c`,
  `indirect-test.c`, `namei-test.c`, and `dir-test.c` covering:
  * Block verified as `EXT4_BLOCK_BITMAP` rejected with `-EFSCORRUPTED` by
    `ext4_xattr_check_block()` (and reverse order: `EXT4_XATTR` rejected by
    `ext4_validate_block_bitmap()`).
  * `EXT4_XATTR` vs. `EXT4_EXTENT` and `EXT4_XATTR` vs. `EXT4_DIR_DATA`
    collisions.
  * HTree `DX_ROOT` self-loop rejected when probed as `DX_NODE`, and valid
    `DIR_DATA -> DX_ROOT` conversion in `make_indexed_dir()`.
  * Freed metadata block (`ext4_forget()`) cleanly reallocated as a different
    metadata type.

#### Commit 19: `ocfs2: enforce metadata block types across ocfs2 validators and cache`

* Extend `ocfs2_read_blocks()` / `ocfs2_validate_*()` / `ocfs2_journal_access_*()`
  and new-block initialization sites with `JBUF_TYPE_OCFS2_{SUPER,INODE,EXTENT,
  GROUP_DESC,DIR_BLOCK,DX_ROOT,DX_LEAF,XATTR,REFCOUNT,QUOTA,SLOT_MAP}` and
  `JBUF_TYPE_DATA` in `fs/ocfs2/aops.c`.
* Call `jbuf_reset_type()` on non-JBD cached buffers in
  `ocfs2_metadata_cache_purge()` (cluster lock downconvert) and
  `ocfs2_remove_block_from_cache()` (local block free) so legitimate
  cross-node block reallocation never causes stale-type false positives.

#### Commit 20: `docs: ext4: document jbuf block cache architecture`

* Commit `Documentation/filesystems/ext4/block_cache.rst` (reconciled with any
  minor API adjustments made during implementation) and add `block_cache` to
  the `.. toctree::` in `Documentation/filesystems/ext4/overview.rst`.
* Verify clean Sphinx documentation build (`make htmldocs`).

---

## 5. Per-Commit Verification Workflow

Following `writing-kunit-tests.md` and `using-kvm-xfstests.md`, every single
commit is gated on the following checklist before advancing to the next commit:

1. **KUnit fast-loop tests (UML, built-in and modular)**:
   ```bash
   # Standalone jbuf KUnit suite (from Commit 1 onward)
   ./tools/testing/kunit/kunit.py run \
       --kunitconfig=fs/jbd2/.kunitconfig \
       --build_dir=<scratch>/kunit-jbuf --jobs=$(nproc)

   # Combined jbuf + all 23 ext4 KUnit suites (built-in)
   ./tools/testing/kunit/kunit.py run \
       --kunitconfig=fs/ext4/.kunitconfig \
       --build_dir=<scratch>/kunit-ext4 --jobs=$(nproc)

   # Modular KUnit export/link check
   ./tools/testing/kunit/kunit.py run \
       --kunitconfig=fs/ext4/.kunitconfig \
       --kconfig_add=CONFIG_MODULES=y \
       --kconfig_add=CONFIG_JBUF_KUNIT_TEST=m \
       --kconfig_add=CONFIG_EXT4_KUNIT_TESTS=m \
       --build_dir=<scratch>/kunit-mod --jobs=$(nproc)
   ```
   Also run newly added test cases individually
   (`kunit.py run "<suite>.<test_case>"`) to confirm strict test isolation.

2. **Full x86_64 kernel builds (zero warnings)**:
   * Appliance kernel with debug/sanitizers (`install-kconfig --lockdep
     --kasan --extra-debug` + `CONFIG_OCFS2_FS=m` so every commit compiles all
     of `jbuf`, `jbd2`, `ext4`, and `ocfs2`).
   * From Commit 2 onward, an additional build with `CONFIG_BUFFER_HEAD=n`
     (disabling legacy buffer-head filesystems) verifying zero stray
     `buffer_head` dependencies in `jbuf`, `jbd2`, or `ext4`.

3. **Style and patch hygiene**:
   ```bash
   ./scripts/checkpatch.pl -g HEAD
   ```
   Confirm zero `ERROR`/`WARNING`/`CHECK` issues and confirm no `Signed-off-by:`
   tag was emitted.

4. **VM filesystem validation (`kvm-xfstests`)**:
   ```bash
   # Full smoke run after every commit (~15-20 min)
   kvm-xfstests --kernel <path/to/bzImage> smoke
   ```
   Check `get-results --failures` for zero regressions and inspect `dmesg` for
   zero lockdep, KASAN, or `jbuf_cache_destroy()` leak warnings. On milestone
   commits (Commits 2, 7, 12, 13, 17, 18), additionally run targeted configs:
   * `-c 4k,1k,nojournal,data_journal,inline,fast_commit,bigalloc_4k,encrypt,quota,ext3,ext3conv -g quick` (or targeted subsets covering external journal, online resize, `buffered_iomap`, and corrupted-image type-mismatch reproducers in `kvm-xfstests shell`).
   * If `mkfs.ocfs2` / `mount.ocfs2` (local-mount mode `-M local`) is
     available in the test environment, run local-mode OCFS2 mount, buffered
     read/write/mmap/truncate, xattr, reflink, quota, fsck, and unmount checks
     in `kvm-xfstests shell`; otherwise explicitly report OCFS2 runtime testing
     status alongside verified clean compilation.

---

## 6. Resolved Design Decisions

1. **Source location (`fs/jbd2/`)**: `jbuf.c`, `jbuf-test.c`, and `.kunitconfig`
   live in `fs/jbd2/` rather than `fs/`, keeping the implementation scoped to
   `jbd2`'s two users (`ext4` and `ocfs2`) while still supporting `ext4` in
   no-journal mode (since `EXT4_FS` unconditionally selects `JBD2`).
2. **Call-site variable names**: Existing `struct jbuf *bh` variable names are
   preserved at call sites in this series to keep diffs focused and easy to
   review; any mechanical variable rename via open-source Coccinelle is
   deferred until after dead-code elimination.
3. **Documentation commit placement**: `Documentation/filesystems/ext4/block_cache.rst`
   and its `overview.rst` entry are committed at the end of the series
   (Commit 20).
4. **`ocfs2` direct I/O scope**: `ocfs2` keeps legacy `__blockdev_direct_IO()`
   in this series; the goal here is for `ext4` to step completely away from
   `CONFIG_BUFFER_HEAD` while keeping `ocfs2` functional across the `jbd2`
   migration.
5. **Phase ordering**: Dead-code elimination (Phase 2, Commits 3–9) runs first,
   followed by `journal_head` unification and struct packing (Phase 3,
   Commits 10–14), and then `b_meta_type` metadata type identification
   (Phase 4, Commits 15–19).
6. **Mounted block-device access & XFS-level loose coherency**: Userspace
   programs (such as bootloaders and `blkid`) retain full read access to a
   mounted block device via `bdev->bd_mapping` (`def_blk_aops`), while writes
   to a mounted block device are not supported by the base series (even
   though the block layer permits them unless `CONFIG_BLK_DEV_WRITE_MOUNTED=n`
   or `bdev_allow_write_mounted=0`; see §3.2 item 6 for the userspace writers
   this affects). Cache coherency matches XFS: `sync_blockdev()` +
   `invalidate_bdev()` at mount, `invalidate_bdev()` at unmount (and after
   online label/UUID superblock ioctls), and no cross-cache invalidation
   overhead during normal mounted operation.
7. **Header name**: The `jbuf` API header is `include/linux/jbd2_jbuf.h`
   (not `include/linux/jbuf.h`), reflecting that it is scoped to `jbd2` and
   its users.
8. **Userspace writes to the mounted block device**: Handling of `e2fsck` on a
   read-only mounted root, boot code in the first fs block, and legacy
   `tune2fs`/`e2label` direct superblock writes is pending approval (§7). If
   approved, it is permanent, because the first two must be supported
   indefinitely.

---

## 7. Optional Plan (Pending Approval): Block-Device Superblock Coherency

> [!IMPORTANT]
> Nothing in this section is part of the base series until approved. Because
> `e2fsck` of a read-only mounted root and boot code written into the first fs
> block must be supported indefinitely, this is a **permanent** mechanism:
> there is no temporary Kconfig option and no removal schedule. Legacy
> `tune2fs`/`e2label` direct superblock writes are handled by the same code.

### 7.1 Requirements and Scope

The following userspace writers must keep working while the file system is
mounted. All of them write through the mounted device node into
`bdev->bd_mapping`, which the kernel's `jbuf` cache does not share (see
§3.2, item 6):

* **A. `e2fsck` of a read-only mounted root, followed by
  `mount -o remount,rw`.** `__ext4_remount()` already re-reads `s_state`
  because "it may have been changed by e2fsck since we originally mounted the
  partition". When `e2fsck` has nothing to repair, it updates `s_state`,
  `s_lastcheck`, `s_mnt_count`, and the error-information fields without
  asking for a reboot.
* **V. Boot code in the first 1024 bytes.** This covers MBR boot code when
  ext4 occupies a whole disk with no partition table (which XFS cannot offer,
  because its superblock is in sector 0), and partition boot records written by
  installers such as `extlinux --install`. For every block size above 1 KiB,
  including the default 4 KiB, these bytes are in fs block 0 together with the
  superblock, and the kernel rewrites the whole block whenever it writes the
  superblock.
* **B. Legacy `tune2fs`/`e2label`**, which write superblock fields directly
  instead of using `EXT4_IOC_SET_TUNE_SB_PARAM`, `EXT4_IOC_SETFSUUID`, or
  `FS_IOC_SETFSLABEL`.

Readers (bootloader tools, `blkid`, `dumpe2fs`) keep read access with
XFS-level freshness: they see the superblock as of the kernel's last write of
it (§7.9).

Out of scope:

* Writes through a different device node (for example, the whole disk when
  ext4 is on a partition), which are not coherent today either.
* Metadata other than the superblock's fs block.
* File systems mounted with `-o sb=`.
* `ocfs2`.

`tune2fs -j` (`-O has_journal`) on a mounted file system is rejected.

### 7.2 Design Overview

* **Userspace copy**: the `bd_mapping` folio that caches the superblock's fs
  block. With `CONFIG_BUFFER_HEAD=y` it carries `def_blk_aops` buffer heads;
  otherwise it carries iomap state. `ext4` never attaches anything to it.
* **Base**: the 1024-byte superblock image that the userspace view (the cached
  copy, or the on-disk block if nothing is cached) is expected to hold. It is
  the image the kernel last wrote or read at the home location, or the
  userspace image it last merged. The previous base is kept for checksum
  validation. Because the kernel refreshes any cached copy whenever it writes
  the block (§7.4), any difference from base is a userspace change.
* **Nothing is pinned.** `ext4` holds a reference to the userspace copy only
  while it checks it. When no copy is cached, nothing is pushed, pulled, or
  merged.
* Three mechanisms:
  1. **Home-write verification** (§7.4) guarantees that the kernel never
     overwrites a userspace change on disk, including an `O_DIRECT` write.
  2. **The probe and worker** (§7.5) make buffered writes take effect promptly
     and evict idle copies.
  3. **Sync points** (§7.5): the getter ioctls, `ext4_statfs()`, read-only to
     read-write remount, unmount, and freeze.

### 7.3 Pinning, Overhead, and Eviction

| State | Kernel work |
| --- | --- |
| No userspace copy cached (the normal case) | One lockless page-cache lookup per probe interval, on a deferrable timer. Nothing else, apart from the §7.4 check when the kernel itself writes the superblock. |
| Copy cached and unchanged (for example, after `blkid` or `dumpe2fs`) | Each probe locks the folio and compares the superblock bytes (and boot-code bytes). The copy is evicted with `invalidate_mapping_pages()` once it has gone `bdev_sb_evict_timeout_ms` (default 30 s) with no change and no kernel refresh. |
| Copy changed by userspace | Validated three-way merge (§7.6); the kernel's next write of the superblock then refreshes the copy. |

Eviction only drops clean, unlocked, unmapped folios, and only after any change
in the copy has been merged, so it never discards userspace data. Per-mount
state is two 1 KiB images, a pending change set, and one delayed work item.

### 7.4 Home-Write Verification (Never Overwrite Userspace Changes)

`s_sbh` is flagged with a new `JBH_PreWrite` bit. `jbuf`'s write-submission
path calls the cache's `prewrite` callback (Commit L2) before every write of
that buffer to its home location: journal checkpoint, writeback,
`ext4_commit_super()`, and `sync_dirty_buffer()`. Writes into the journal
itself are not affected. The submitter holds the buffer lock, and the `ext4`
callback:

1. Gets the userspace view with `read_mapping_folio()` on `bd_mapping`. On a
   cache hit there is no I/O. On a miss it reads the block from disk. This is
   the only I/O the design adds, and it is what catches `O_DIRECT` writes and
   buffered writes whose copy was already written back and evicted.
2. Locks the copy and waits for any writeback on it to finish. If the
   superblock bytes equal base and the boot-code bytes equal the buffer's,
   there is nothing to do.
3. Otherwise, merges into the buffer: it adopts the boot-code bytes (V),
   applies the validated userspace field changes and their in-memory side
   effects under the policy for the current mount state (§7.6), keeps the
   kernel's values for ignored and rejected fields, and recomputes
   `s_checksum`.
4. Copies the final superblock bytes into the cached copy (without dirtying it)
   and makes them the new base, so the copy matches what is about to be written.
5. If anything was merged, queues the change set for the worker. The worker
   journals it, so that a journal replay cannot reapply an older logged copy of
   the block, and updates the backup superblocks.

The copy that step 1 reads stays cached until the probe evicts it, so:

* the read happens at most once per eviction period, and
* userspace reads during the kernel's write are served from the refreshed copy
  rather than from disk.

The kernel writes the superblock's block at these times:

* mount, unmount, remount, and freeze/thaw;
* after the periodic `s_sb_upd_work` refresh (at most hourly by default);
* after journaled superblock changes: `s_last_mounted` on first open, lazily
  set feature flags, the tuning/label/UUID ioctls, error recording, and
  orphan-list head changes on file systems without `orphan_file`.

The callback runs in process context (the jbd2 checkpoint, the flusher, or the
caller of `ext4_commit_super()`), and does its read under
`memalloc_noio_save()`. If the read fails, the kernel writes its own image and
warns once.

### 7.5 Probe, Worker, and Sync Points

* **Probe**: delayed work on `system_power_efficient_wq` that runs every
  `bdev_sb_poll_interval_ms` (default 5000; 0 disables probing). It calls
  `filemap_get_folio()`, which does no I/O. If no copy is cached, it is done.
  Otherwise it locks the copy and compares it. If the copy changed, it runs the
  worker; if the copy has been idle for the eviction timeout, it evicts it.
* **Worker** on a read-write mount:
  * Applies the pending change set through `ext4_update_superblocks_fn()`.
    That function journals the primary and two backups, updates the remaining
    backups, and refuses to run during online resize (`EXT4_FLAGS_RESIZING`).
  * Then makes the same in-memory updates as `ext4_ioctl_set_tune_sb()`.
  * For boot-code changes, journals block 0 so that the newest logged copy
    carries the new bytes.
  * If the change set holds only ignored or rejected fields, still journals the
    primary so that the kernel's values are written back.

  The primary reaches its home location at the next checkpoint (or immediately
  in no-journal mode), going through §7.4.
* **Worker** on a read-only mount: merges in memory only, using the read-only
  policy (§7.8).
* **Worker scheduling**: the worker uses `sb_start_intwrite_trylock()`, and
  keeps the change set queued while the file system is frozen, shut down, or
  resizing. Base is updated when a change set is queued, and change sets hold
  absolute values. So re-applying a change set is idempotent, and nothing is
  merged twice.
* **Sync points**:
  * `FS_IOC_GETFSLABEL`, `EXT4_IOC_GETFSUUID`, and `EXT4_IOC_GET_TUNE_SB_PARAM`
    run a verification (which may read the block) and, when possible, run the
    worker synchronously. New tools therefore see changes made by old ones,
    even with `O_DIRECT`.
  * `ext4_statfs()` only does the lookup and kicks the worker, so `df`
    reflects `tune2fs -m` promptly.
  * Read-only to read-write remount runs a forced verification (§7.8).
  * Unmount and freeze write the superblock, so they go through §7.4.

### 7.6 Merge Policy and Field Classification

| Class | Fields | Treatment |
| --- | --- | --- |
| **Merged** (tuning ioctl) | `s_errors`, `s_mnt_count`, `s_max_mnt_count`, `s_checkinterval`, `s_lastcheck{,_hi}`, `s_r_blocks_count_{lo,hi}`, `s_def_res{uid,gid}{,_hi}`, `s_default_mount_opts`, `s_def_hash_version`, `s_raid_stride`, `s_raid_stripe_width`, `s_mount_opts`; feature bits being *set* within `EXT4_TUNE_SET_{COMPAT,INCOMPAT,RO_COMPAT}_SUPP`; `EXT4_ERROR_FS` newly set in `s_state` (treated as `EXT4_TUNE_FL_FORCE_FSCK`) | Validated and applied with the `EXT4_IOC_SET_TUNE_SB_PARAM` helpers (Commit L1) |
| **Merged** (other ioctls) | `s_volume_name`; `s_uuid` | As `FS_IOC_SETFSLABEL`; as `EXT4_IOC_SETFSUUID` (as with the ioctl, `sb->s_uuid` changes only at the next mount) |
| **Merged** (persist-only) | `s_last_mounted` (`tune2fs -M`) | Copied; no runtime effect |
| **Merged only on read-only mounts** (`e2fsck`, A) | In `s_state`: `EXT4_VALID_FS` set and `EXT4_ERROR_FS` cleared; `s_error_count` and the `s_{first,last}_error_*` fields cleared | Error fields are accepted only if the kernel has recorded no newer error since base |
| **Ignored** (kernel-owned) | `s_free_blocks_count{,_hi}`, `s_free_inodes_count`, `s_wtime{,_hi}`, `s_mtime{,_hi}`, `s_kbytes_written`, other `s_state` changes, `s_last_orphan`, `needs_recovery`/`orphan_present`, error fields (on read-write mounts), `s_overhead_clusters`, `s_encrypt_pw_salt`, `s_checksum` | The kernel's value wins silently (several of these change on every write by e2fsprogs) |
| **Rejected** | Everything else: geometry and layout; revision and creator fields; feature changes outside the set masks (all clears, `has_journal`, …); journal fields (`s_journal_inum`, `s_jnl_blocks`, …), so `tune2fs -j` is rejected; `s_hash_seed`; quota/orphan-file/lost+found inode numbers; `s_encoding{,_flags}`; MMP fields; `s_checksum_type`/`s_checksum_seed`; `s_flags`; `s_{min,want}_extra_isize`; reserved bytes | Dropped with one warning per field per mount, and the kernel's values are written back. On read-only mounts they are recorded instead (§7.8). |
| **Boot code** (block size above 1 KiB) | Every byte of the superblock's fs block outside the 1024-byte superblock | Owned by userspace and always adopted, because the kernel never modifies these bytes |

* The classification is a `static const` table of
  `{offset, size, class, tune flag}` entries built with
  `offsetof()`/`sizeof_field()`. A KUnit test asserts that the entries cover
  all 1024 bytes exactly once. Any field added to `struct ext4_super_block`
  later therefore fails the test until someone classifies it.
* With `metadata_csum`, the userspace `s_checksum` must match one of two images:
  the copy as it stands, or the copy's changed bytes replayed onto the previous
  base. The second case covers a kernel superblock write that lands between the
  tool's read and its write. If neither matches, the write is rejected with a
  warning.
* The first time a direct write is merged on a read-write mount, an
  informational message (once per mount) names the fields and suggests the
  ioctl-based tools. Adopting boot code is silent.

### 7.7 Userspace Access Paths, Including `O_DIRECT`

All of the paths below go through the mounted device node (`block/fops.c`).

| Access | Block-layer behavior | Today (`buffer_head`) | With §7 |
| --- | --- | --- | --- |
| Buffered read | `filemap_read()`; `blkdev_read_folio()` on a miss | Served from the pinned `s_sbh` page, so the reader sees the in-memory superblock | Served from the copy, which holds the on-disk superblock as of the kernel's last write; the probe evicts the copy after the timeout |
| Buffered write or `mmap` store | Dirties the copy, which `fsync()` or the flusher writes back; the copy stays cached | Lands in `s_sbh` immediately, unvalidated | Merged by the probe within one interval; §7.4 also catches it, without I/O, before any kernel write |
| `O_DIRECT` read | `kiocb_write_and_wait()` flushes dirty cached pages in the range, then reads the disk | On-disk superblock | Same |
| `O_DIRECT` write | `kiocb_invalidate_pages()` flushes and invalidates the range, the data goes to disk, then `kiocb_invalidate_post_direct_write()` runs. If invalidation returns `-EBUSY`, `blkdev_direct_write()` returns 0 and `blkdev_write_iter()` falls back to buffered I/O | `try_to_free_buffers()` fails on the busy `s_sbh` buffer head, so the write silently becomes a buffered write into `s_sbh`'s page | Goes to disk and leaves no cached copy, so the probe cannot see it. The next §7.4 check misses the cache, reads the disk, and merges the change before the kernel writes; a getter ioctl, remount, or unmount does the same. The change is never overwritten, and takes effect at the next of these events |

* The block layer does not order a kernel write and an `O_DIRECT` write of the
  same block issued at the same moment. Whichever reaches the device last wins,
  as with any two unsynchronized writers. (Today's buffered fallback serialized
  them on the page lock.)
* `mmap` stores are not atomic with respect to the comparison. A torn update
  fails checksum validation (with `metadata_csum`); otherwise it is merged as
  seen.

### 7.8 Read-Only Mounts and `e2fsck` (A)

* While the file system is mounted read-only, the probe merges `e2fsck`
  updates into the in-memory superblock with the read-only policy, without
  writing anything. Rejected changes are recorded, not overwritten.
* On read-only to read-write remount, `__ext4_remount()` runs a forced
  verification (from the cache or the disk) with the read-only policy. This
  happens before the existing `ext4_clear_journal_err()` call and
  `s_mount_state` re-read.
  * If a rejected change is recorded or found, the remount fails with a message
    asking for a reboot, as `e2fsck` does after repairing a mounted root.
  * Otherwise the remount proceeds, and `ext4_setup_super()` writes the merged
    superblock through §7.4.
* As today, repairs to other metadata of a mounted root still need the reboot
  that `e2fsck` asks for, because the kernel's cached metadata is stale.

### 7.9 Locking and Lifecycle

* **Lock order**: journal handle → `lock_buffer(sbi->s_sbh)` →
  `s_bdev_sb->lock` → the copy's folio lock.
  * The §7.4 callback is entered with `s_sbh` already locked by the submitter.
    From writeback, the `jc_mapping` folio is also locked, which comes before
    `s_sbh` in the order.
  * Nothing starts a handle, writes `s_sbh`, or waits for a commit while
    holding `s_bdev_sb->lock` or the copy's lock.
  * Apart from this code, only page-cache and block-device paths take the
    copy's lock, and they never call into `ext4`.
* **Mount**: after journal replay, set base from `s_sbh`, invalidate any clean
  cached copy of the block (it may predate the replay), flag `s_sbh`, and start
  the probe. The superblock write in `ext4_setup_super()` is then the first
  verified write.
* **Unmount**: cancel the delayed work. The final `ext4_commit_super()` is
  verified like any other write. Because nothing is pinned, nothing has to be
  released before `jbuf_cache_destroy()` and `invalidate_bdev()`.
* **Disabled** when `CONFIG_BLK_DEV_WRITE_MOUNTED=n`, or when mounted with
  `-o sb=`.
* **Read freshness**: readers see the on-disk superblock as of the kernel's
  last write of it. Counters are refreshed at most hourly by `s_sb_upd_work`
  (or by `fsfreeze`), and journaled changes appear after checkpoint. Pushing
  the in-memory superblock into cached copies would restore today's live view,
  but it complicates base tracking, so it is not proposed.

### 7.10 Configuration (sysfs)

These are per-file-system attributes in `/sys/fs/ext4/<dev>/`, documented in
`Documentation/ABI/testing/sysfs-fs-ext4`. Writing either one re-arms the
delayed work.

| Attribute | Default | Meaning |
| --- | --- | --- |
| `bdev_sb_poll_interval_ms` | 5000 | Probe interval. 0 disables probing, so changes take effect only at the sync points and §7.4 checks. Nonzero values below 100 are raised to 100. |
| `bdev_sb_evict_timeout_ms` | 30000 | How long a cached copy may go without a change or kernel refresh before it is evicted. 0 evicts it at the first opportunity, so every kernel superblock write then reads the block. |

### 7.11 Alternatives Considered

* **Pinned sentinel folio** (the previous revision of this section): detects
  `O_DIRECT` writes without reads, because `folio_unmap_invalidate()` detaches
  the pinned folio without checking its reference count. But it keeps a
  block-device page cached for the life of the mount and pushes on every
  superblock change. It is replaced by on-demand copies plus §7.4 verification.
* **Block-layer write notification**: a `blk_holder_ops` callback from
  `blkdev_write_iter()` for writes that overlap a range the holder has
  registered. It is event-driven and needs neither the probe nor verification
  reads. Now that the mechanism is permanent it is a reasonable follow-up, but
  it needs block-layer and VFS changes.
* **`e2fsck` using `EXT4_IOC_SET_TUNE_SB_PARAM`** for its bookkeeping updates,
  with a small set of fields allowed on read-only mounts: A's merge policy would
  then only be a fallback for older e2fsprogs. V would still need this
  infrastructure.
* **Aliasing `s_sbh` onto the `bd_mapping` folio**, for exact legacy
  semantics: rejected. It requires `jbuf`/`jbd2` to handle a folio they do not
  own, and it reintroduces unvalidated live mutation of `s_es`.
* **Two-way diff without base**: rejected, because it cannot tell userspace
  changes from the kernel's own.

### 7.12 Commit Plan (If Approved)

Recommended placement: immediately after Commit 2, so that no bisectable point
overwrites these userspace writes. Existing commit numbers are left unchanged
here to keep the review simple.

* **L1**
  **`ext4: separate superblock tuning validation from its side effects`**:
  * Split `ext4_validate_tune_sb_params()` into a pure validator and an apply
    step that makes the `sbi` updates now done during validation
    (`s_csum_seed`, `s_hash_seed`, `s_hash_unsigned`, `s_def_hash_version`).
  * Factor the label and UUID checks out of `ext4_ioctl_setlabel()` and
    `ext4_ioctl_setuuid()`.
  * No functional change; extend `fs/ext4/ioctl-test.c`.
* **L2** **`jbd2: jbuf: add a pre-write callback for flagged buffers`**:
  * Add the `JBH_PreWrite` bit, and add to `struct jbuf_cache` an optional
    `jc_ops->prewrite()` callback with a `jc_private` cookie (the `ext4` side
    stores `sbi` there). The callback is called with the buffer locked, before
    a flagged buffer is submitted for writing.
  * KUnit: the callback is called for writes only, may modify the data before
    submission, and is not called for unflagged buffers or jbd2 shadow buffers.
  * Commit 7 must keep calling it from `jbuf_meta_write_folio()`.
* **L3** **`ext4: keep the superblock coherent with block device writes`**:
  * `fs/ext4/bdev_sb.c`: base tracking, the §7.4 callback, the probe and
    eviction, the worker, the sync points, the field table with the read-write
    policy (B), the sysfs attributes and their ABI documentation, and the
    warnings.
  * KUnit suite `fs/ext4/bdev_sb-test.c`, with the page-cache accessors
    redirected through `KUNIT_STATIC_STUB_REDIRECT`. Cases:
    * no copy cached, so no work is done;
    * an unchanged copy is evicted after the timeout;
    * a buffered change is merged by the probe;
    * an `O_DIRECT`-style change (no cached copy, and the disk differs) is
      merged by the callback before the write;
    * the checksum race is accepted;
    * a rejected field is restored;
    * the field table covers the whole superblock;
    * sysfs values are clamped.
* **L4** **`ext4: honor e2fsck superblock updates at ro-to-rw remount`**: the
  read-only policy, recorded rejections, and remount refusal, with KUnit tests.
* **L5** **`ext4: preserve boot code in the superblock's fs block`**: adopt the
  non-superblock bytes in the callback and journal them from the worker, with
  KUnit tests.
* Commit 20 adds a "Userspace writes to a mounted device" subsection to
  `block_cache.rst`, and documents the sysfs attributes in
  `Documentation/admin-guide/ext4.rst`.

### 7.13 Testing

* KUnit as above, plus a build with `CONFIG_BLK_DEV_WRITE_MOUNTED=n`.
* `kvm-xfstests smoke`, plus scripted scenarios in `kvm-xfstests shell`. The
  test device `/dev/vdc` holds a whole-disk ext4 file system, which is exactly
  the V configuration.

**Tools:**

* Buffered writers: `debugfs -w -R "ssv ..."`, which always writes through
  `write_primary_superblock()`, and a pre-ioctl `tune2fs`/`e2label` copied in
  through `/vtmp`.
* `O_DIRECT` writers and readers: `xfs_io -d -c "pwrite ..."` and
  `dd iflag=direct`/`oflag=direct`.
* Cache state: `fincore /dev/vdc`.

**Scenarios:**

1. **Idle**: with no userspace access, page 0 of `/dev/vdc` is never cached,
   and superblock writes (`fsfreeze`, unmount) still work. After `blkid`, the
   copy disappears once `bdev_sb_evict_timeout_ms` has passed.
2. **Buffered merge**: merged fields are visible through the getter ioctls and
   `df` within one probe interval. They persist across an unmount, and across
   `xfs_io -x -c shutdown` followed by a remount.
3. **`O_DIRECT` writes** of a merged field and of boot-code bytes: superblock
   writes (`fsfreeze -f`/`-u`, unmount) do not revert them, and they are
   visible after the next superblock write or getter call.
4. **Ignored and rejected fields**: kernel-owned fields are ignored, and
   rejected fields (including `tune2fs -j`) are restored; `e2fsck -fn` is
   clean.
5. **A**: read-only mount, `e2fsck -fy`, then `mount -o remount,rw`. The
   updates survive, and an injected rejected change makes the remount fail.
6. **V**: write a pattern into bytes 0–445 (the MBR boot code area) with both
   buffered and `O_DIRECT` I/O, generate superblock writes, then unmount. The
   pattern must be intact on disk, including after a forced shutdown and
   journal replay.
7. **sysfs**: set `bdev_sb_poll_interval_ms=0`, and try small values for both
   attributes.
8. **Stress**: `fsstress` running alongside repeated legacy writes; 1 KiB block
   size; `nojournal`.

Consider upstreaming scenarios 2–6 as new xfstests `ext4/` tests.

### 7.14 Open Questions for Approval

1. Approve this design for merging into the main plan, placed right after
   Commit 2?
2. Is the `jbuf` pre-write callback (L2) acceptable as the only change to the
   `jbuf` core?
3. Is the verification read when no copy is cached acceptable? It is at most
   one block read per eviction period, and only while the kernel is writing the
   superblock. The alternative is the block-layer write notification.
4. Should B's read-write merge policy be kept indefinitely, with the
   once-per-mount informational message? It shares all of A's and V's code.
5. Is it acceptable that readers such as `dumpe2fs -h` on a mounted file
   system see the on-disk superblock rather than the in-memory one?
6. Are the attribute names and units (`bdev_sb_poll_interval_ms`,
   `bdev_sb_evict_timeout_ms`) acceptable?

Resolved in this revision:

* The mechanism is permanent.
* `tune2fs -j` on a mounted file system is rejected.
* The probe interval defaults to 5 s and the eviction timeout to 30 s; both
  can be set through sysfs.
