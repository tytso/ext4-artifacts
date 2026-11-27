# e2fsprogs: evaluation of the open GitHub issues

Evaluated against https://github.com/tytso/e2fsprogs/issues on 2026-11-22, updated on 2026-11-26 after closing 14 already-fixed issues, and updated on 2026-11-27 after `maint` (`cf354a9d`) was published to GitHub (and merged through `f3c16ef2` into `master`/`next` at `76b5934a`) and the 2 newly fixed issues (#219, #259) plus 8 superseded pull requests (#141, #142, #270, #274, #283, #287, #289, #301) were closed: **70 open issues** remain (down from 86 originally, 72 after 2026-11-26; plus **37 open pull requests**, down from 45, read only where they bear on an issue or overlap `maint` commits). No changes were made to the e2fsprogs repository.

## How to read this report

**Two published branches were checked, because `maint` and `master` still diverge.**
- **`maint` (`cf354a9d`, published to `github/maint` and `origin/maint`):** carries **78 commits** beyond the previous `origin/maint` (`10884526`) — all 61 commits that were previously on `pu` (`10884526..2080baa6`) plus **17 new commits** (`2080baa6..cf354a9d`: `libsupport/compat.c` for macOS deprecation warnings and a `badblocks` I/O refactor, `-S` seed option, scalable in-memory simulator `tst_badblocks_sim` with 21 `b_*` regression tests, and bug-fix series).
- **`master` == `next` (`76b5934a`, published to `github/master` and `github/next`, matching `origin/master` and `origin/next`):** merged `maint` at `f3c16ef2` (`Merge branch 'maint' into next`). It therefore includes all 61 former-`pu` commits and the first 15 of the 17 new `maint` commits, plus the 97 `master`-only commits (the fuse2fs rework, `fuse4fs/`, libsupport cache code, a few libext2fs changes).
- **Remaining divergence:** merge base between `github/maint` (`cf354a9d`) and `github/master` (`76b5934a`) is now `f3c16ef2`. Only 2 commits on `github/maint` are not yet in `github/master`: `aeb114fb` (`libsupport: fix -Werror build failures in compat_daemon()`) and `cf354a9d` (`badblocks: report out-of-range numeric arguments consistently on 32-bit`). On GitHub, `github/pu` (`66fe42db`) points to an older v1.46.3-era orphan-file branch. Where nothing is said below, `maint` and `master` have identical code.

**Verification levels** used in the entries:

| Tag | Meaning |
|---|---|
| **Re-run** | Reproduced by an evaluator and then re-run by me against out-of-tree builds of `pu` (now `maint`) and/or `master` |
| **Reproduced** | Reproduced once, by the evaluator for that area; I checked the cited code or commit but did not re-run it |
| **Read** | Settled by reading the code; nothing was executed |
| **History** | Settled from git history (the fixing commit's diff was read) |
| **Unverified** | Could not be checked here; the entry says what would settle it |

Every commit hash below was confirmed with `git`, including which of a release tag, `github/master` (`76b5934a`) and `github/maint` (`cf354a9d`) contain it. "First release" is the earliest `vX.Y.Z` tag containing the commit.

**What could not be done here:** nothing that needs root, a mount, a real block device, Windows/macOS/Cygwin/musl/BSD/Android toolchains, or the network beyond the GitHub issue text (so attachments, gists, lore links and off-site patches referenced from issues were not read). fuse2fs was not mounted and e4defrag was not run. The local builds lack libarchive, so tar-input behaviour was exercised with the distribution's mke2fs 1.47.4, whose `create_inode_libarchive.c` is byte-identical to HEAD.

## Summary

| | Count | Issues |
|---|---|---|
| **High** | 4 | #145, #108, #295, #282 |
| **Medium** | 14 | #111, #292, #98, #117, #174, #139, #302, #37, #242, #255, #97, #199, #134, #299 |
| **Low** | 52 | everything else still open |

- **Closed on 2026-11-27 (2 issues, 8 PRs — fixed/superseded on `github/maint` and `github/master`):** issues #219, #259 and PRs #141, #142, #270, #274, #283, #287, #289, #301 (verified closed on GitHub; archived in [Closed since initial triage](#closed-since-initial-triage) below).
- **Closed on 2026-11-26 (14 issues — already fixed in released versions):** #56, #81, #85, #89, #92, #96, #120, #158, #173, #178, #235, #240, #254, #264 (verified closed on GitHub; archived in [Closed since initial triage](#closed-since-initial-triage) below).
- **Already supported, only undocumented (1):** #130.
- **Duplicates still open:** #175 → #111; #33 → #199; #179 and #176 → #217; #16 → #38; #155 → #300.
- **Open PRs that are correct fixes for a still-open issue:** #303 (for #286), #304 / #281 (for #271 — #304 has cleaner style; both still omit `lib/e2p/getflags.c`), #263 (for #255, with changes), #285 (for #282, with changes).

### High priority

| # | Problem | Verified | Why High |
|---|---|---|---|
| #145 | `e2fsck` reports a valid file with preallocation past EOF and an extent tree of depth ≥ 2 as corrupt; `-y` deletes the file, `-p` fails the check | Re-run, `maint` and `master` | e2fsck destroys data on a consistent file system; boot-time `-p` check fails |
| #108 | When `e2fsck` empties a non-root extent node it then fails with "No 'up' extent" and clears the whole inode | Re-run (same reproducer) | Turns a repairable (or, with #145, non-existent) problem into loss of the whole file |
| #295 | Inline-data symlink with an external xattr block: `e2fsck -fy` frees the in-use xattr block; `resize2fs -M` then aborts part-way and leaves damage | Re-run, `maint` and `master` | Two tools each damage a consistent file system |
| #282 | Inline-data size taken from the `system.data` xattr is never bounded; consumers assume it fits in a block (heap overflow in `e2fsck` pass 2 and in libext2fs) | Read (not executed) | Memory corruption in e2fsck/libext2fs from a crafted image, on every branch |

### Medium priority

| # | Problem | Verified | Disposition |
|---|---|---|---|
| #111 | `e2fsck -c` / `mke2fs -c` ignore a failed `badblocks` run; `e2fsck -c` then discards the existing bad-block list | Reproduced | Fix (tracking issue for #175) |
| #292 | `resize2fs -P`/`-M` on bigalloc returns roughly the whole device (cluster/block unit mix-up) | Re-run | Fix (3 lines) |
| #98 | Offline grow by more than the reserved-GDT horizon fails and leaves the file system needing e2fsck | Reproduced | Fix |
| #117 | Growing past the GDT limit without `meta_bg` is still refused | Reproduced (scaled down); online half unverified | Finish the partial fix |
| #174 | New finding inside a support thread: flex_bg minimum-size estimate 1.2–1.7× too large | Reproduced | Fix carefully, then close as answered |
| #139 | tdb scratch file wraps at 4 GiB and overwrites its own header; e2fsck crashes hours in | Root cause reproduced standalone | Fix |
| #302 | `tune2fs -E mount_opts=a,b` cannot be expressed | Re-run | Fix |
| #37 | Changing the UUID of a mounted fs prints "run e2fsck -f" instead of "must be unmounted" | Read | Fix (reorder) |
| #242 | fuse2fs permission checks wrong for create and unlink in the default mode | Read | Fix |
| #255 | `mke2fs -d <tar>` drops `security.selinux` | Reproduced (distro binary) | Merge PR #263 with changes |
| #97 | No `ext2fs_rm`/`ext2fs_rmdir`; the duplicated copies have diverged, and `debugfs rm` leaks xattr blocks | Leak re-run | Feature, plus a real bug fix |
| #199 | `make rpm` has been unbuildable since 2017 | Reproduced (simulated) | Repair the spec or remove the target |
| #134 | `m_assume_storage_prezeroed` fails as root on a non-tmpfs `$TMPDIR` | Read | Fix the test |
| #299 | `NOTICE` says lib/ext2fs is LGPL; several file headers say otherwise | Read | Maintainer decisions needed |

### Already supported or partially fixed (still open)

| # | Commit | First release | In `github/master` / `github/maint` | Action |
|---|---|---|---|---|
| #130 | `2e8ca9a2` (2004) io-layer URL syntax, `image?offset=N` (works; undocumented) | long-standing | yes / yes | Document, then close as answered |

Partially fixed: #189 (`44850e62`, `bf7996b9`, `bb75f4e9`, `26ff7213`, in `maint` and `master`, `badblocks -i` half; `e2fsck -c -k` half still open), #211 (`ebc3ff15`, in `maint` and `master`, `num_blocks` consistency; `first_block` range percentage still open), #117 (`ad56ccaa`, v1.47.2), #63 (`37c2008f`, v1.46.2, uid/gid half), #198 (`064b8da0`, v1.47.2, leak item), #133 (`386e7db5`, v1.47.3, libarchive only), #91 (functionality exists; documentation missing).

---

## Per-issue evaluation (70 open issues)

Each entry gives: priority, verification level, finding, assessment of any proposed fix, and a design sketch. Original line numbers are from the 2026-11-22 evaluation at `2080baa6` (now in `maint`/`master`) unless updated below for the `2080baa6..cf354a9d` `maint` series. The 16 issues closed on 2026-11-26 and 2026-11-27 have been moved to [Closed since initial triage](#closed-since-initial-triage).

### A. e2fsck correctness and memory safety

#### #145 — fsck breaks preallocated files — **High**
- **Status:** valid, not fixed on `maint` or `master`. **Re-run.**
- **Finding:** a file with `i_size` larger than one block, unwritten blocks preallocated past EOF, and an extent tree of depth ≥ 2 is reported as "end of extent exceeds allowed value". Cause: libext2fs synthesizes the length of the last index entry in each node from `i_size` (`lib/ext2fs/extent.c`, `end_blk`), `scan_extent_node()` passes that down as the child's bound, and the past-EOF exemption added by `085757fc` (v1.42.9) only applies to leaf extents carrying the uninit flag. Index entries never carry it.
- **My re-run:** on a clean image holding 400 one-block unwritten extents (depth 2), changing only `i_size` to 3072 gives five such errors; `e2fsck -fp` stops with UNEXPECTED INCONSISTENCY; `e2fsck -fy` ends by clearing the inode and removing the directory entry. The same layout at depth 1 stays clean. Identical on `master`.
- **Caveat:** the images were built with `debugfs`/libext2fs, not written by a kernel. The layout is what `fallocate(FALLOC_FL_KEEP_SIZE)` produces, but a loop-mounted kernel test would close that gap. The evaluator's estimate that about 170 GiB of contiguous preallocation is needed on 4k blocks (about 11 GiB on 1k) is a calculation, not a measurement.
- **Proposed fix:** the reporter's patch is in a gist that was not readable here; by their description it changes the library's `end_blk` computation and is incomplete for sparse files. Changing the library value would alter `ext2fs_extent_get()` output for index entries (visible in `debugfs ex` and many expect files), so I would not take that route.
- **Design:**
  - Fix in `e2fsck/pass1.c:scan_extent_node()` only. Make the bound passed to a child structural: next sibling's start − 1 for a non-last index entry, the node's own bound for the last one (0 = unbounded at the root).
  - Do not apply `PR_1_EXTENT_END_OUT_OF_BOUNDS` to a last index entry, whose length is synthetic.
  - This changes behaviour for initialized extents past EOF on the right-hand edge (they fall through to the `i_size` check, as at depth 0 today). Run all `f_extent_*`, `f_uninit_ext_past_eof*`, verity tests and `j_fc_extent_overflow`.
  - Tests: `tests/f_prealloc_deep_tree/` built with `mke2fs` + `debugfs set_bmap --uninit` (expect a clean fsck), plus a variant with data inside EOF.
  - Land together with #108.

#### #108 — e2fsck: Error in deleting parent extent — **High**
- **Status:** valid, not fixed on `maint` or `master`. **Re-run** (same reproducer as #145, and a second one by the evaluator with initialized extents).
- **Finding:** when `scan_extent_node()` deletes every entry in a non-root node, `ext2fs_extent_delete()` also removes the empty node and its parent entry and leaves the handle one level up. The recursive caller then does `EXT2_EXTENT_UP` again, gets `EXT2_ET_EXTENT_NO_UP`, and `check_blocks_extents()` raises `PR_1_EXTENT_ITERATE_FAILURE`, whose fix is to clear the inode. Undamaged leaves are lost with it.
- **Proposed fix:** none; the report's diagnosis is accurate.
- **Design:**
  - In `scan_extent_node()`, after the recursive call returns, call `ext2fs_extent_get_info()`; if the handle is already at the caller's level the child was removed, so skip the `UP`, skip `mark_block_used()` and the block count for that node, and resume as the existing delete path does. If the level is lower still, return so the next caller up repeats the check.
  - No library ABI change. Alternative: `EXT2_EXTENT_DELETE_KEEP_EMPTY` and remove the node explicitly from the parent invocation.
  - Also worth fixing: in `ext2fs_extent_delete()` the return value of the recursive delete is overwritten by the following `ext2fs_write_inode()`.
  - Test: `tests/f_extent_leaf_emptied/` (programmatic image), with a depth-2 variant.

#### #295 — Inline symlink with external xattr causes fsck bitmap errors and resize2fs failure — **High**
- **Status:** valid, not fixed on `maint` or `master`. **Re-run** on both builds with the reporter's exact exit codes (0 / 4 / 1 / 0 / 1 / 12; the control without `inline_data` is all 0).
- **Finding:** two independent defects.
  1. `e2fsck/pass1.c` (the `LINUX_S_ISLNK` branch, around line 2019): for a symlink with `EXT4_INLINE_DATA_FL` pass 1 does `FINISH_INODE_LOOP(); continue;` without calling `check_blocks()`. That is the function that accounts for `i_file_acl`, so the xattr block is never marked in use; pass 5 reports it as a bitmap difference and `-y` frees a block that is still referenced.
  2. `lib/ext2fs/valid_blk.c:ext2fs_inode_has_valid_blocks2()`: for a symlink with an EA block it returns 1 when `i_size >= 60` before it reaches the `EXT4_INLINE_DATA_FL` test, so `resize2fs` calls `ext2fs_block_iterate3()` on an inline-data inode and aborts mid-resize.
- **Proposed fix:** the reporter's suggestion (test inline data first in `valid_blk.c`) is right for defect 2 and, as they observed, cannot affect defect 1.
- **Design:**
  - `pass1.c`: call `check_blocks(ctx, &pctx, block_buf, &ea_ibody_quota)` for inline-data symlinks before `FINISH_INODE_LOOP()`, as the fast-symlink branch below already does.
  - `valid_blk.c`: move the `EXT4_INLINE_DATA_FL` test above the symlink heuristics. This also corrects the other callers (e2fsck pass 2 and `super.c`, `e2image`, `tune2fs`, quota code).
  - An evaluator tried exactly these two changes in a scratch copy: the reproducer becomes clean at every stage and the suite shows one expected change, a new and correct "reference count 1, should be 2" line in `tests/f_inlinedata_repair/expect.1` (that image has an inline symlink sharing an EA block that e2fsck ignores today). That experiment is evidence the sketch is sound, not a reviewed patch.
  - Tests: `tests/f_inline_symlink_ea_block/` and `tests/r_inline_symlink_ea_block/`, both programmatic.
  - Two commits (`e2fsck:` and `libext2fs:`), both candidates for `maint`.
- **Side effect of defect 1:** a corrupt `i_file_acl` on an inline-data symlink is also never diagnosed by pass 1 today.

#### #282 — Unbounded ext4 inline_data_size causes heap out-of-bounds write in two independent e2fsck paths — **High**
- **Status:** valid by code reading on `maint` and `master`; not fixed. **Read: neither the reporter, the evaluator nor I executed it.** I confirmed the two cited sites myself.
- **Finding:** `ext2fs_inline_data_size()` returns `EXT4_MIN_INLINE_DATA_SIZE + ea_size` with no upper bound (`lib/ext2fs/inline_data.c:111`), and `ext2fs_inline_data_get()` copies that many bytes into the caller's buffer. The kernel and the libext2fs write path only ever keep `system.data` in the inode body, but the read path accepts whatever the generic xattr reader returns, which with `ea_inode` can be far larger than a block. Consumers assume it fits: `e2fsck/pass2.c:1158` computes `fs->blocksize - inline_data_size` in `size_t`, and `ext2fs_inline_data_convert_dir()` copies into a one-block buffer. None of the inline-data commits on `maint` adds a bound.
- **Additional sink not in the report:** `lib/ext2fs/fileio.c` copies inline data into the three-block `file->buf` on the ordinary file read/write path, so `debugfs cat`/`dump` and fuse2fs are exposed as well, not only e2fsck.
- **What is not established:** the exact failure mode, and whether some earlier check rejects such an image first. The evaluator looked in pass 1 (`check_ea_in_inode()`, `get_inline_data_ea_size()`) and in `read_xattrs_from_buffer()` and found none. An ASAN run on a suitable test image, before and after a fix, would settle it.
- **Proposed fix (PR #285):** correct as far as it goes. Its check in `ext2fs_inline_data_ea_get()` sits at the single choke point and bounds all consumers. Gaps: the bound (one block) is looser than the real invariant (the value must fit in the inode body and must not live in an EA block or EA inode); the `convert_dir` check ignores the 12-byte checksum tail under `metadata_csum`; there is no e2fsck-side detection or repair, so e2fsck would handle the new library error only incidentally; no regression test; tested only by "a fresh image still fscks clean".
- **Design:**
  - libext2fs: reject an oversized `system.data` in `ext2fs_inline_data_ea_get()` using the tighter bound `EXT2_INODE_SIZE - EXT2_GOOD_OLD_INODE_SIZE`; better, add an internal helper (declared in `ext2fsP.h`, no ABI change) that refuses a `system.data` that is not an in-body, non-EA-inode attribute.
  - Keep PR #285's defensive check in `ext2fs_inline_data_convert_dir()`, subtracting `csum_size`.
  - e2fsck pass 1: treat an over-bound size like a broken attribute, via the existing `PR_1_INLINE_DATA_NO_ATTR` truncate path or a dedicated `PR_1_*` code. Decide deliberately whether that should be automatic under `-p`, since it truncates data (PR #268 proposes making that code preen-safe).
  - e2fsck pass 2: belt-and-braces `inline_data_size > fs->blocksize` check before the `memset()`.
  - Tests: an `f_` test (a small prebuilt `image.gz` is justified here because the entry has to be written raw) and a `tst_inline_data` case asserting `ext2fs_inline_data_size()` returns an error.
- **Disposition:** merge PR #285 with those changes.

#### #139 — e2fsck fails using scratch_file — **Medium**
- **Status:** valid, not fixed. Root cause **Reproduced** with a standalone tdb program; the e2fsck crash itself was not re-run (it needs a multi-billion-inode file system).
- **Finding:** `tdb_off_t`/`tdb_len_t` are 32-bit (`lib/ext2fs/tdb.c:125-126`) and `tdb_expand()` has no overflow check. When the scratch file reaches 4 GiB it is truncated to the wrapped size and filled with pad bytes over its own header, which matches the reporter's evidence exactly.
- **Priority rationale:** a crash hours into a run, but only with opt-in `[scratch_files]` on very large file systems, and the damage is confined to the scratch file.
- **Design:**
  - `tdb_expand()` / `tdb_expand_file()`: detect 32-bit overflow and fail with `TDB_ERR_OOM`.
  - e2fsck: turn that into a clear "scratch file limit of 4 GiB reached" message in both users (`lib/ext2fs/icount.c`, `e2fsck/dirinfo.c`).
  - Avoid the limit up front: do not choose tdb when the directory count implies more than 4 GiB.
  - Document the limit in `e2fsck.conf.5`. Test: make the limit a variable in `tdb.c` so a unit test can lower it.

#### #271 — fgetflags assigns uninitialized value to *flags (when ioctl fails) — **Low**
- **Status:** valid (**Read**), harmless for in-tree callers, which all test the return value first. Same on all branches.
- **Proposed fix (PR #281, PR #304):** both are correct and minimal for `lib/e2p/fgetflags.c`. PR #304 (diff read) uses cleaner K&R style (`} else\n\t*flags = f;`) than PR #281, though neither fixes the identical pattern in `lib/e2p/getflags.c`.
- **Design:** `if (r == -1) { ... } else *flags = f;` in both files; no ABI change, no test needed. **Disposition:** merge PR #304 (extending it to `lib/e2p/getflags.c`) and close PR #281.

#### #286 — ext2fs_check_directory_fuzzer.cc calls open() with O_CREAT but no mode — **Low**
- **Status:** valid (**Read**); `tests/fuzz/ext2fs_check_directory_fuzzer.cc:26`. The two sibling fuzzers pass `0644`.
- **Proposed fix (PR #303):** adds `, 0644`. Correct and complete. **Disposition:** merge PR #303. Note that oss-fuzz carries its own copy of the file.

#### #228 — Uninitialized Variable Bug in e2fsck/emptydir.c — **Low**
- **Status:** valid but dead code (**Read**). `e2fsck/emptydir.c` is not compiled; its only reference is `po/POTFILES.in`. The bad test came from `70303df1` (v1.45.6).
- **Design:** delete the file and its `POTFILES.in` line, as already stated in the issue. **Disposition:** fix by deletion, then close.

#### #227 — Inappropriate Use of realloc … (contrib/fsstress.c) — **Low**
- **Status:** valid but inconsequential (**Read**). Nothing builds the file since `efe9817e` (2017) removed it from the Android build.
- **Design:** delete `contrib/fsstress.c` and point at the canonical copies in xfstests/LTP. **Disposition:** close: won't fix.

#### #188 — Checking the source code with a static analyzer — **Low**
- **Status:** item 1 (the `memset()` before `free()` in `misc/e4crypt.c:clear_secrets()` may be elided) is a valid hardening nit with no realistic exposure, since the process exits immediately afterwards. Item 2 (a `badblocks` cleanup loop "never runs") is a **false positive**: the analyzer did not model `setjmp` returning twice, and `num_saved` is deliberately `static`. **Read.**
- **Design:** answer and close. Optional: `AC_CHECK_FUNCS(explicit_bzero)` and use it in `clear_secrets()`.

#### #198 — oss-fuzz targets need improvement — **Low**
- **Status:** item 2 (leak from repeated `add_error_table()`) fixed by `064b8da0` (v1.47.2; all branches). Items 1, 3 and 4 remain open enhancements. **History/Read**; fuzzers not built.
- **Design:** for coverage, add `-fsanitize=fuzzer-no-link` to the library CFLAGS under `--enable-fuzzing`; put harness logging behind an `#ifdef`; a harness that walks a file system (inode scan, directory and block iteration, xattr read) would exercise the paths behind #282 and PRs #287/#288. **Disposition:** keep open as a feature request, noting item 2 is fixed.

#### #119 — fsck task blocked for more than x seconds — **Low**
- **Status:** cannot determine. A D-state hang on kernel 4.9 with eMMC; nothing ties it to e2fsprogs. **Disposition:** close as stale/needs info, pointing at the board or kernel vendor.

#### #115 — e2fsck: Specify what is "rewritten" on a Force rewrite action — **Low**
- **Status:** partially valid as a usability/documentation point; answered in the thread. **Reproduced:** journal replay still runs before the device-size comparison.
- **Design:** document "Ignore error" / "Force rewrite" in `e2fsck.8`; in `e2fsck/ehandler.c` do not offer the rewrite when the block is beyond the device; in `e2fsck/unix.c` compare sizes before touching the journal; an `f_` test with a truncated image and a dirty journal. **Disposition:** document, plus the small code change.

#### #78 — Is there an error code description for e2fsck? — **Low**
- **Status:** a usage question, answered at length by the maintainer; reporter silent since 2021. **Disposition:** close: answered. Optional: one sentence in `e2fsck.conf.5` saying the codes are listed in `e2fsck/problem.h`.

#### #59 — Please add a "Quit Safely" command to the e2fsck interactive mode — **Low**
- **Status:** mostly exists but is not discoverable: `a` = yes-to-all (`aeb933b3`, v1.43) and Ctrl-C cancels cleanly with exit 32. No `q` answer. **Read.**
- **Design:** accept `q` in `e2fsck/util.c:ask_yn()` and treat it like Ctrl-C; show the choices once per run; on cancel after changes were written, clear `EXT2_VALID_FS` so the next boot re-checks; document in `e2fsck.8`. **Disposition:** keep open: feature request.

### B. resize2fs

#### #292 — Resize fail with bigalloc feature, and tune2fs output problem — **Medium**
- **Status:** valid, not fixed. **Re-run:** an empty 4 GiB bigalloc file system reports an estimated minimum of 986584 of 1048576 blocks on both `maint` and `master`.
- **Finding:** `calculate_minimum_resize_size()` (`resize/resize2fs.c:3112-3125`) subtracts per-group free counts, which are in clusters, as if they were blocks. Regression from `ac94445f` (v1.45.5); the clamp added by `265f8890` (v1.46.5) is in the wrong unit too. The remark that `tune2fs -l` has no "Cluster count" line is not a bug; no such field ever existed.
- **Priority rationale:** `-P`/`-M` are useless and plain shrink is refused on every bigalloc file system; limited to bigalloc, and `-f` works around it (while also disabling the safety check).
- **Design:** make `n` a `blk64_t`, clamp against `EXT2_CLUSTERS_PER_GROUP()`, convert with `EXT2FS_C2B()` before use. An evaluator's scratch patch doing this passed the 34 resize-related tests and produced clean, data-preserving `-M` results. Review point: the bigalloc estimate then becomes tight (two free clusters left in one populated test) and fragmented bigalloc was not tested. Test: `tests/r_bigalloc_min_size/`.

#### #98 — resize2fs fails if new size is more than 1024 times larger than initial size and large inodes are used — **Medium**
- **Status:** valid, not fixed. **Reproduced** with the issue's script on both builds.
- **Finding:** the resize inode's double-indirect block is relocated beyond the old end of the file system, and `lib/ext2fs/block.c` rejects it while iterating through the old handle. The aborted run leaves inode 7 pointing past the end; `e2fsck -fy` repaired it with no data loss in the evaluator's test.
- **Priority rationale:** deterministic failure that leaves the file system flagged with errors, but repairable, before inode tables move, and with an unusual geometry. Two independent reports from factory-flashing users suggest it is not rare in embedded work.
- **Design:**
  - Narrow option: in `inode_scan_and_fix()`, for non-extent inodes, raise the old handle's block count to the new size only around the `ext2fs_block_iterate3()` call. A scratch patch doing this fixed the reproducer and passed the 34 resize tests.
  - **Do not** raise it for the whole inode scan: that variant crashed `r_32to64bit_expand_full` with heap corruption (not debugged). Whatever is chosen should be run under ASAN.
  - Cleaner, library-side option: a new `BLOCK_FLAG_*` bit for `ext2fs_block_iterate3()` to relax the post-callback range check (new flag, no signature change).
  - Test: no small-image reproducer was found; the issue's 64M → 65G sparse geometry takes about 4 s and 4.6 MB, or mark it slow.
- **Unverified:** that ordinary block-mapped files are affected the same way (code reading only; only the resize-inode case was demonstrated).

#### #117 — Error when running resize2fs on a filesystem from 180TB to 360TB? — **Medium**
- **Status:** partially fixed by `ad56ccaa` (v1.47.2; all branches), which skips the descriptor-count check only when `meta_bg` is already set. A file system without `meta_bg`, which is the reporter's case, still gets the error. **Reproduced** scaled down for the offline case.
- **Unverified:** online behaviour (needs root), and the suggested two-step online workaround, which rests on recollection of kernel behaviour. The lore patch linked from the issue was not read.
- **Design:** in `resize/main.c` apply the check only for offline resize (`online_resize_fs()` already rejects what old kernels cannot do); make the offline message actionable ("resize it while mounted"). Larger follow-up, as a feature: let offline resize enable `meta_bg` when the GDT no longer fits (`adjust_fs_info()`). Test: `tests/r_gdt_limit_no_meta_bg/` for the offline half.

#### #174 — Correct smallest size estimation for resize2fs shrink — **Medium** (Low as literally filed)
- **Status:** as filed this is a feature request ("make `-M` optimal") that the maintainer already answered. While checking it an evaluator found a concrete defect. **Reproduced.**
- **New finding:** on flex_bg file systems the loop that accumulates `last_start` (`resize/resize2fs.c:3194-3219`, from `45a78b88`, v1.42.10) never runs, because `flex_groups` was already rounded up past the new group count. The capacity of the added groups is then counted again as "data blocks in last group". Six flex_bg samples gave minimums 1.2–1.7× larger than needed (for example 297434 vs 178706 blocks); without flex_bg the numbers match.
- **Design:** recompute `last_start` after the loop as the sum of usable blocks over the groups before the last; keep the deliberate flex_bg slack. **This needs far broader validation than it got** (seven samples, contiguous unwritten extents, no fragmentation): an under-estimate makes `-M` fail part-way, which is the outcome worth avoiding most. Cover fragmented files, 1k/2k blocks, stride, bigalloc, sparse_super2, meta_bg and repeated `-M`. Then reply with the existing advice and close.

#### #191 — TRIM remainder after shrinking? — **Low**
- **Status:** feature request; resize2fs has no discard code. `blkdiscard --offset` after the shrink covers it today.
- **Design:** opt-in flag; after a successful offline shrink call `io_channel_discard()` on `[new_size, old_size)`; treat `EOPNOTSUPP` as a note; never discard after a failed resize; refuse or warn with `-z undo_file`. **Disposition:** keep open: feature request.

#### #20 — Superblock / partition table mismatch – misleading error message formulation — **Low**
- **Status:** wording of `PR_0_FS_SIZE_WRONG`, unchanged since 1997. The reporter's proposed text is wrong for non-partition devices.
- **Design:** reword only the string in `e2fsck/problem.c` to name both possible causes (device shrunk or not enlarged, or corrupt superblock), keeping the code and flags. No expect file contains the text. **Disposition:** small rewording, or close as answered.
- **Separate observation (Reproduced):** on a file system larger than its device `e2fsck -fn` prints the warning and then exits 0, while `-fy` exits 8 and `-p` exits 4. A script using `e2fsck -n` as a health check would miss it.

### C. mke2fs

#### #255 — Allow setting selinux xattrs when creating a filesystem from a tar — **Medium**
- **Status:** valid, not fixed: tar input copies only `security.capability` and `gnu.translator` (`misc/create_inode_libarchive.c:473`). **Reproduced** with the distribution binary.
- **Proposed fix (PR #263):** functionally correct and applies to both branches. Needs: a name table (or prefix rule) instead of a longer `strcmp` chain, a test, a man page sentence and a commit message.
- **Design:** decide the policy rather than adding one name at a time. Copying every xattr present in the archive is defensible because the user put them there and `-E no_copy_xattrs` is the opt-out; check how libarchive presents ACLs before widening that far. Test by extending `tests/m_rootpaxtar`. **Disposition:** merge PR #263 with changes.

#### #249 — mke2fs -t ext4 fails on kernel 6.12.49 — **Low**
- **Status:** support request, answered. The underlying fragile kernel-version parse is still present. **Reproduced** with a faked `uname` release: an out-of-range sublevel makes mke2fs pick revision 0 and fail with "Journals not supported with revision 0 filesystems".
- **Design:** harden `parse_version_number()` (use `strtoul`, clamp the sublevel, treat failure as "unknown"); a failed parse must never select revision 0. Consider deleting the pre-2.2 and pre-2.6 fallbacks outright. **Disposition:** small fix, then close as answered.

#### #95 — mke2fs can spin forever when creating very large (e.g. 256PB) file systems — **Low**
- **Status:** valid, not fixed. **Reproduced** by the evaluator with a debugger backtrace in the `ipg_retry` loop of `ext2fs_initialize()` (`lib/ext2fs/initialize.c:372`). My own check only shows `mke2fs -n` at 64 PiB still running after 20 s on both builds, which is consistent but not conclusive.
- **Priority rationale:** a hang without data loss at sizes nobody creates; but it is an unbounded loop in a library API with a few-line fix.
- **Design:** if `8 * group_desc_count` exceeds 2^32, return `EXT2_ET_TOO_MANY_INODES`; make the retry loop self-terminating by never stepping `ipg` below 8; optionally a friendlier mke2fs message suggesting bigalloc. Test: `mke2fs -n` at 64 PiB expecting an error, under `timeout`.

#### #238 — mke2fs has no way to directly set the default mount count — **Low**
- **Status:** valid feature request; `tune2fs -c` afterwards is the workaround.
- **Design:** `max_mount_count` and `check_interval` relations in `mke2fs.conf` next to `enable_periodic_fsck`, optionally mirrored as `-E` options; validate against the `__s16` field. **Disposition:** keep open: feature request.

#### #91 — mke2fs cannot (easily) be made deterministic — **Low**
- **Status:** the capability exists (`e1f71006` `hash_seed`, v1.44.0; `b6e29130` and `f353a1f1` `SOURCE_DATE_EPOCH`, v1.47.1) and was verified to give bit-identical images on both branches. It is undocumented in the man pages.
- **Design:** document `SOURCE_DATE_EPOCH`/`E2FSPROGS_FAKE_TIME` and a "reproducible images" recipe in `mke2fs.8`, with the caveats (timestamps are clamped, not set; `-d <dir>` inherits host ownership and xattrs). Optional `m_reproducible` test. **Disposition:** document.

#### #76 — mkfs: feature to output filesystem to stdout — **Low**
- **Status:** not a defect; the layout needs seekable output. **Disposition:** close: won't fix, pointing at sparse image files and `-d -` for streamed input.

#### #163 — Is there a way to pass selinux file_contexts file to mkfs.ext4? — **Low**
- **Status:** question. No `file_contexts` support exists; the options are a pre-labelled `-d` tree, `-E root_selinux=` (`9fc251d0`, v1.47.4), or `debugfs ea_set`. **Disposition:** close: answered, cross-referencing #255.

### D. badblocks

`github/maint` (`cf354a9d`) now carries a 15-commit `badblocks` series (`f2192d6f`..`ebc3ff15`, plus `cf354a9d`), of which the first 14 commits (through `ebc3ff15`) are also merged into `github/master` (`76b5934a`). The series factors device I/O behind `struct bb_dev_ops` (`f2192d6f`), adds `-S <seed>` reproducible random seeding (`14646a5e`), adds a scalable in-memory device simulator (`tst_badblocks_sim`, `d7827229`) with **21 deterministic `tests/b_*` regression tests** (`6a39edc1` and follow-up commits), and fixes a suite of `badblocks` bugs — including a heap buffer overflow in `test_nd()` after a short read (`42a73b63`), the `#219` write-mode corruption bug (`b2caa055` + `9c99ce5e`), `-i` known-bad skipping and batch recovery (`44850e62`, `bf7996b9`, `bb75f4e9`, `26ff7213`), 32-bit wraparound near `2^32 - 1` (`13889d17`), and consistent `num_blocks` across phases (`ebc3ff15`). Only `cf354a9d` (consistent 32-bit out-of-range error reporting in `parse_uint()`) is `maint`-only so far. `e2fsck/badblocks.c` and `misc/mke2fs.c:test_disk()` were not touched; they still prepend `/sbin` to `PATH`.

#### #111 — Badblocks fails on really large disks, can take e2fsck down as well — **Medium**
- **Status:** partially valid, still open. The 32-bit block limit (> 2^32 blocks) is deliberate and has a workaround (`-b 4096`). The "e2fsck hangs" claim was not reproduced; an 8 TB scan is most likely just slow. Note: `13889d17` (in `maint` and `master`, test `b_large_offset`) fixed 32-bit block-number wraparound infinite loops *near* `2^32 - 1`, and `cf354a9d` (`maint` only) made `>= 2^32` numeric errors consistent on 32-bit platforms, but the core 32-bit limit and `e2fsck -c`/`mke2fs -c` defects below are unchanged.
- **Real bug found (Reproduced):** e2fsck and mke2fs ignore badblocks' exit status. When badblocks fails, `e2fsck -c` wipes the existing bad-block list and carries on, and `mke2fs -c` exits 0 having tested nothing.
- **Design:**
  - Check the `pclose()` status in `read_bad_blocks_file()` and mke2fs `test_disk()`; abort before `ext2fs_update_bb_inode()` on failure. This must be coordinated with PR #236, which changes badblocks' exit code.
  - Refuse `-c` up front when the file system has more than 2^32 blocks; the bad-block inode cannot record such blocks anyway.
  - Make badblocks' error suggest `-b`; optionally 64-bit support for stand-alone use.
  - Build the child command safely (see "Additional defects").
- **Disposition:** fix; keep as the tracking issue and close #175 into it.

#### #175 — badblocks fails on large drives due to 32 bit value — **Low**
- Duplicate of #111. **Disposition:** close, noting `-b 4096` (or larger above 16 TiB).

#### #189 — badblocks to ignore — **Low**
- **Status:** partially fixed (`badblocks` side done; `e2fsck` side still open). The entire `badblocks -i` bug suite (PR #141) is now **fixed on both `github/maint` and `github/master`** (more thoroughly than PR #141, with 4 regression tests): `44850e62` (restore batch size after known-bad clamp, `b_ro_read_errors`), `bf7996b9` (don't treat block 0 as end-of-list, `b_known_bad_list`), `bb75f4e9` (skip `-i` blocks in `-w` mode, `b_rw_known_bad`), and `26ff7213` (multi-pattern `bb_iter` rewind and `bb_count` preservation in `test_nd()`, `b_nd_multipat`). However, `e2fsck -c` (`e2fsck/badblocks.c`) still never passes the known bad-block list to `badblocks` via `-i`, so known bad blocks are re-tested every time.
- **Proposed fix:** the reporter's gitlab patch was not readable. PR #141 was closed on 2026-11-27 as superseded by `44850e62` + `bf7996b9` + `bb75f4e9` + `26ff7213`.
- **Design:** in `e2fsck/badblocks.c:read_bad_blocks_file()`, when keeping the old list (`-k`), write it to a temp file and pass `-i` to `badblocks`. Do not add "skip unreadable metadata" to e2fsck. **Disposition:** keep #189 open, narrowed to the `e2fsck -c -k` half.

#### #211 — badblocks: suggestions — **Low**
- **Status:** partially addressed; still open. `ebc3ff15` (in `maint` and `master`, test `b_verbose_status`) made `num_blocks = last_block` consistent across all phases. However, `print_status()` (`misc/badblocks.c:294`) still computes `calc_percent(currently_testing, num_blocks)` ignoring `first_block`, so the percentage is still wrong when a non-zero first block is given (**Read** on `maint`, previously **Reproduced**). The cursor-hiding request is cosmetic.
- **Design:** compute the percentage over `[first_block, last_block]` and add a `tst_badblocks_sim` test; decline cursor hiding. **Disposition:** fix the range-relative percentage only.

#### #207 — badblocks: test_pattern size discrepancy — **Low**
- **Status:** still open: the man page (`misc/badblocks.8.in:183`) still says `ULONG_MAX-1`, while the code caps at 32 bits (`UINT_MAX`; `cf354a9d` on `maint` also made out-of-range `parse_uint()` error reporting consistent on 32-bit platforms and for values ≥ 2^64). **Design:** correct `misc/badblocks.8.in` (value up to 0xffffffff, at most four bytes) and optionally document the byte layout. **Disposition:** document.

#### #190 — badblocks – text could be improved for write tests — **Low**
- **Status:** still open (fair wording point; `misc/badblocks.c:513` still prints `"Testing with pattern 0x"` for `-w`). **Design:** print `"Writing pattern 0x..:"` from `test_rw()` and keep `"Testing with pattern"` where it is accurate (updating `b_verbose_status` / simulator tests). **Disposition:** fix.

#### #113 — FR: badblocks – allow supplying percentages for first_block and last_block — **Low**
- **Status:** still open; partly covered since v1.41.7 by `83bfa27d` ("Interrupted at block N"). `-w` is not truly resumable from one number because it makes eight passes.
- **Design:** show the current block in the status line; accept a trailing `%`; document. **Disposition:** keep open: feature request.

#### #77 — Badblocks not detecting problems on a known to be failing USB memory stick — **Low**
- **Status:** cannot determine; no badblocks defect explains it. The reporter found `-B` as a workaround. **Disposition:** close as stale, with a man page note that some devices only fail under buffered or larger I/O and that the patterns cannot detect misdirected writes or fake-capacity devices.

### E. tune2fs, dumpe2fs, debugfs

For the files these issues touch `maint` and `master` are effectively identical, and every reproducer was run against both builds with the same result.

#### #302 — `tune2fs -E 'mount_opts=opt1,opt2'` is not correctly parsed — **Medium**
- **Status:** valid, not fixed. **Re-run:** two options give the usage error and exit 1; one option works.
- **Finding:** `-E` is split on commas before `mount_opts` is seen (`misc/tune2fs.c:parse_extended_opts()`). Workaround, confirmed: `debugfs -w -R 'ssv mount_opts a,b'`.
- **Proposed fix:** no patch. The reporter's "mount_opts must be last and swallows the rest" works but changes the meaning of `mount_opts=discard,stride=8`, which sets both today. Their alternative of `-E x -E mount_opts=…` does not work as things stand because only the last `-E` is honoured.
- **Design:** pick one of (1) greedy to end of string, simple but a behaviour change, or (2) accept `mount_opts="a,b"` with a quote-aware tokenizer, fully compatible. Either way make repeated `-E` accumulate, make the "too long" path an error, update the man page and usage text. Test: `tests/t_mount_opts/`.

#### #37 — Misleading message when trying to change UUID (^metadata_csum_seed, mounted fs) — **Medium** (low end)
- **Status:** valid, not fixed (**Read**; the mounted path cannot run without root). In the `-U` block `check_fsck_needed()` runs before the mounted test, so a mounted file system is told to "run e2fsck -f" and never sees the accurate "must be unmounted / enable metadata_csum_seed" text.
- **Priority rationale:** the message steers users toward running e2fsck on a mounted file system and hides the remedy; the fix is a reorder.
- **Design:** move the mounted test ahead of `check_fsck_needed()`; refuse the `ea_inode`-without-`metadata_csum` case on a mounted file system with the same message. The mounted case cannot have a fail-before test in the suite; say so in the commit message.

#### #90 — debugfs: Exit status is 0 even if some errors occurred — **Low**
- **Status:** valid. **Re-run:** exit 0 for a failed command and even for a failed open.
- **Design:** install a `com_err` hook in `debugfs` `main()` that sets a global "error seen" flag (about 230 call sites go through `com_err()`), and fold it into the exit status for `-R`/`-f`; treat a failed open of a device named on the command line as non-zero. Expect-file churn is the cost; gate behind a flag if that is too disruptive. **Disposition:** keep open: feature request.

#### #130 — FR: Ability to set offset= in debugfs for disk images in files — **Low**
- **Status:** already supported via `debugfs 'image?offset=BYTES'` (`2e8ca9a2`, 2004). **Reproduced** for read and write. It is undocumented in every man page. **Disposition:** document, then close as answered. Optional `tests/d_offset/`.

#### #79 — debugfs: add warning to manpage about mounted filesystems — **Low**
- **Status:** valid; `debugfs.8` has no such warning and `debugfs -w` never calls `ext2fs_check_if_mounted()`. **Design:** add the warning under `-w`; optionally warn (not refuse) at open time. **Disposition:** document.

#### #63 — Support for extended attributes and big UID/GID using debugfs rdump — **Low**
- **Status:** the uid/gid half was fixed by `37c2008f` (v1.46.2). Restoring xattrs/ACLs is still missing, and neither PR #135 nor PR #266 provides it.
- **Design:** an opt-in `rdump -x` that iterates the inode's xattrs and applies them with `lsetxattr()`, reporting `security.*`/`trusted.*` failures once. **Disposition:** keep open, retitled to the xattr half.

#### #100 — Difference in output of 'dumpe2fs -h' versus 'tune2fs -l' — **Low**
- **Status:** valid observation; `tune2fs -l` omits the journal-superblock block. **Design:** document it. If the output is wanted in tune2fs, share the helpers via `misc/util.c`, but make them warn-and-continue rather than `exit(1)`, since `tune2fs -l` is often used on damaged file systems. **Disposition:** document.

#### #195 — tune2fs has inconsistent output when setting different fs options — **Low**
- Cosmetic (**Reproduced**). Any change churns `t_*` expect files. **Disposition:** close: won't fix, or lowest-priority feature request.

#### #99 — Some program IDs and usages are printed to STDOUT instead of STDERR — **Low**
- Valid inconsistency, but moving the tune2fs banner would break scripts that strip the first line of `tune2fs -l`. **Disposition:** close: won't fix (historical, kept for compatibility).

#### #156 — [feature request] ability to specify undo-file in config — **Low**
- **Status:** valid request; e2fsck creates an undo file only with `-z`. **Design:** an `[options]` boolean in `e2fsck.conf` that sets `ctx->undo_file` to the default name when not read-only; document the cost; test alongside `u_e2fsck_opt`. Also clarify `e2fsck.conf.5`, whose `undo_dir` text reads as if undo files were created by default. **Disposition:** keep open: feature request.

### F. fuse2fs, e4defrag, filefrag, libext2fs API

fuse2fs differs substantially between branches: `maint` has the older `misc/fuse2fs.c`; `master` has the reworked one plus `fuse4fs/`. Nothing was mounted, and e4defrag was not run, so this section rests on code reading unless stated.

#### #242 — fuse2fs checks permissions incorrectly — **Medium**
- **Status:** valid, not fixed (**Read**) on `maint` fuse2fs, `master` fuse2fs and `master` fuse4fs. Create runs the mode check against the file it has just created, so `open(O_CREAT|O_WRONLY, 0444)` fails; unlink checks `W_OK` on the victim instead of only on the parent. The earlier permission commits (`de86c43f`, v1.47.3; `bc76e0f7`) do not cover this.
- **Priority rationale:** breaks `git clone`, `touch` under a restrictive umask and `rm` of read-only files in the default unprivileged mode; `-o fakeroot` is a workaround; no data loss.
- **Design:** split `check_inum_access()` so callers can ask for the inode-flag part only; for unlink/rmdir/rename use that on the child and keep `W_OK` (add `X_OK`) on the parent, and add the sticky-bit rule, which exists nowhere today; for create, open the just-made inode without the mode check. Apply to `misc/fuse2fs.c` on both branches and `fuse4fs/fuse4fs.c` on `master`. Test: a FUSE test that skips unless `/dev/fuse` and `fusermount3` work.

#### #97 — RFE: libext2fs: ext2fs_rmdir and ext2fs_rm — **Medium**
- **Status:** valid request, not implemented; there are four duplicated implementations (five on `master`).
- **Why Medium:** the copies have already diverged into a real bug. **Re-run:** after `debugfs rm` of a file with an external xattr block, `e2fsck -fn` reports "Block bitmap differences" for that block. The evaluator also saw an unattached EA inode after `rm` with `ea_inode`. Same code on `maint` and `master`.
- **Design:** new `ext2fs_remove_inode()` taken from the most complete copy (fuse2fs `remove_inode()`: release EA inodes, `ext2fs_free_ext_attr()`, punch, free the inode), with `ext2fs_rm()` and `ext2fs_rmdir()` on top. New symbols only; add to `debian/libext2fs2t64.symbols`. Convert debugfs, fuse2fs, the libarchive path and fuse4fs. Test: `debugfs rm`/`rmdir` on objects with an xattr block and an EA inode, expecting a clean fsck (fails today).

#### #300 — [RFE/Proposal] Improve e4defrag fragmentation-scoring algorithm — **Low**
- **Status:** the analysis is valid; the evaluator reproduced the reporter's numbers with a copy of the formula. No patch. The score is report-only and does not gate defragmentation. PR #262 changes which files are defragmented, not the score.
- **Design:** keep existing output lines stable; base the verdict on average extent size against a fixed target; add seek-distance lines to `-c -v`; document the formula in `e4defrag.8`. **Disposition:** keep open: feature request.

#### #155 — e4defrag: Fragmentation score 0 even with files listed with more fragmentation than ideal count — **Low**
- Working as coded: the score is relative to total blocks and printed with `%.0f`. **Disposition:** close: not a bug; track under #300. Optional `%.1f`.

#### #137 — e4defrag freezes in D+ state — **Low**
- A kernel-side hang; nothing in e4defrag can cause it. Which kernel fix applies could not be determined. **Disposition:** close as stale, pointing at linux-ext4 with blocked-task stacks.

#### #138 — Fakeroot like behavior for mounts inside a fuse2fs mounted filesystem — **Low**
- Not a fuse2fs bug: FUSE's `allow_other` rule versus the set-uid `fusermount` (kernel side from memory, unverified). **Disposition:** close: not a bug, with an optional man page note.

#### #136 — fallocate: How to quickly create / convert unwritten extents into written extents? — **Low**
- A question, answered by the maintainer. **Disposition:** close: answered.

#### #88 — Use filefrag -sv to fingerprint COW Reflink extents — **Low**
- The `st_dev` idea was rejected in the thread. PR #132 changes default output and is not mergeable as is. **Design:** an opt-in machine-readable mode. **Disposition:** keep open, narrowed.

#### #93 — RFE: libext2fs: better "how to do this" documentation — **Low**
- Valid: the texinfo manual has no mention of `ext2fs_open2`, `bmap2`, extents, xattrs, punch or fallocate. **Design:** a short cookbook of 8–10 recipes pointing at in-tree reference code; the `tst_libext2fs` harness (now on both `maint` and `master`) is a compact source of call sequences. **Disposition:** document.

#### #30 — RFE: take a BSD file lock on block devices to avoid races against udev probing — **Low**
- **Status:** not implemented. The `io_channel_flock()` infrastructure (`44ce8e42`, `e9dad24e`) is on `master` only, is used only for fuse image files, and locks the partition rather than the whole disk, which is what the udev protocol requires. The patch linked from the thread was not readable.
- **Design:** mke2fs only, Linux only, default off; resolve the whole disk through sysfs, `flock(LOCK_EX)` it for the duration of the run. **Disposition:** keep open: feature request.

### G. Build and portability

For every file these issues touch, `maint` and `master` are identical (`8456fafc` and `aeb114fb` added `libsupport/compat.c` for macOS `sbrk`/`daemon` deprecation warnings and Windows `compat_daemon`, which does not affect the issues below). No MinGW, Cygwin, macOS, ppc64le, musl, BSD or Android toolchain was available, so this section is code reading and history.

#### #217 — Issue when compiling from source on MSYS2 UCRT64, with duplicates #179 and #176 — **Low**
- **Status:** valid, not fixed. `lib/ss` uses `fork`/`wait`/`sigprocmask`/`SIGCONT` unconditionally and the top-level Makefile always builds it. Only the libraries minus libss, plus mke2fs, are supported on Windows (that is what the CI job builds).
- **Design:** add a "Building on Windows" section to `INSTALL` with the CI recipe; better, have `configure` drop `lib/ss` and reduce the program list on `mingw*` so a plain `make` works. **Disposition:** document under #217; close #179 and #176 as duplicates.

#### #133 — Use of dlopen on macOS — **Low**
- **Status:** valid, not fixed for libmagic and readline; the same problem was fixed for libarchive only by `386e7db5` (v1.47.3).
- **Proposed fix (PR #252):** right approach, three defects. **Reproduced:** with pkg-config finding libmagic the feature is silently disabled, and `lib/config.h.in` is not updated so e2fsck fails to link as submitted. From the diff: it reuses the `ARCHIVE` pkg-config prefix, which would contaminate the libarchive check.
- **Disposition:** merge PR #252 with those fixed, or reimplement by copying the libarchive logic (about 40 lines of `configure.ac`).

#### #104 — Having autoconf script check for libexecinfo when looking for backtrace — **Low**
- **Status:** valid gap: `backtrace()` is only looked for in libc, so it is silently compiled out on BSD and musl. The snippet in the issue comment is wrong (it would make configure fail on glibc).
- **Design:** `AC_CHECK_LIB(execinfo, backtrace, …)` into a `BACKTRACE_LIB` added only to e2fsck's link lines. **Disposition:** fix (about 10 lines); close PR #123 against it.

#### #127 — e2fsdroid doesn't build on ppc64le — **Low**
- The failing header is the Android-only `util/android_types.h` used by a third-party CMake build on glibc; the autoconf build derives types differently. **Disposition:** close: not a bug (optional 5-line hardening).

#### #38 — add e2fsdroid compile, with duplicate #16 — **Low**
- Valid feature request with no patch in six years; needs libsparse, libcutils and libselinux. **Disposition:** keep #38 open as the tracking issue; close #16 against it.

### H. Packaging, test suite, licensing

#### #199 — rpms do not build — **Medium** (low end); duplicate #33 — **Low**
- **Status:** valid, not fixed. `make rpm` cannot succeed anywhere: the bundled `e2fsprogs.spec` still lists the ext4dev links removed by `0e7a081c` (first tag containing it: v1.43.4). **Reproduced** by a simulated spec build and a `%files`-versus-install-tree comparison; `rpmbuild` itself was not available. The patch attached to the issue was not readable.
- **Design:** the maintainer has to choose between (a) minimally repairing the spec (drop ext4dev entries, stop packaging what util-linux owns, add e2scrub/e4crypt/e4defrag/fuse2fs, move `make check` to `%check`) and (b) removing `e2fsprogs.spec`, `contrib/build-rpm` and the `rpm:` target. Per-distro specs in `contrib/` was the stated preference but needs volunteers. **Disposition:** fix either way, and close #33 with it.
- **Unexplained `m_rootdir_acl` failures on Rocky (unverified):** `mke2fs -d` copies every host xattr and `tests/filter.sed` drops only `security.selinux`, so running as root on XFS (which exposes `trusted.SGI_ACL_*`) would plausibly fail. Settling it needs a root run on XFS.

#### #134 — Tests failed: m_assume_storage_prezeroed (when /tmp is not a tmpfs) — **Medium** (low end)
- **Status:** valid, not fixed; the test is unchanged since `bd2e72c5` (v1.47.0). It compares an exact sector count that depends on the host file system. It only runs as root, so it was not run here; the explanation of the expected numbers is derived, not measured.
- **Design:** replace the exact value with a bounded check, as the script already does for its first measurement. While there: the script uses `exit 0` although it is sourced, and prints an unset `$DESCRIPTION`.

#### #299 — Licensing issues ext2fs — **Medium**
- **Status:** valid as a factual inventory, with corrections. `NOTICE` says lib/ext2fs is under the Library GPL v2. At HEAD, 9 files there say an unversioned "GNU Public License" (4 with Red Hat or IBM copyright), 9 carry GPL text (`crc16.*`, `crc32c.c`, `rbtree.*`, `nls_utf8.c`, `utf8n.h`, `kernel-jbd.h`, `tdbtool.c`), `fast_commit.h` has an SPDX GPL-2.0 tag, and about 15 have no statement. Corrections to the report: `tdbtool.c` and `sha256.c` are not linked into the library.
- **This needs your decisions, not engineering.** Options, which are not mutually exclusive:
  - Files whose only copyright is yours: relabel as in `543547a5` (2010).
  - Third-party "GNU Public License" files: ask the holders and record consent in the commit, following the `ab00fdac` precedent.
  - Kernel-derived GPL files linked into the library: seek relicensing, replace them, or state the exceptions explicitly in `NOTICE` the way `debian/copyright` already does.
  - Mechanical clean-up that helps under any choice: SPDX identifiers everywhere, fix the dangling "See the file COPYING" references, a header-check script.
- Open PR #161 would add more kernel-derived GPL code to lib/ext2fs and should be weighed against whichever choice is made. I am reporting facts and options here, not legal conclusions.

#### #86 — License is not clear — **Low**
- Valid observation, already answered in the thread: 28 e2fsck files say an unversioned "GNU Public License". **Disposition:** fold into the #299 clean-up, or close as answered.

#### #32 — readlink -e is not portable and fails with busybox — **Low**
- **Status:** valid, not fixed. **Reproduced:** current BusyBox still has no `readlink -e`.
- **Design:** replace the call in `scrub/e2scrub_all.in` with a bash-only `test -e` loop (the script already requires bash). **Disposition:** fix (trivial), or close as won't-fix pointing at the configure options that suppress the units.

---

## Additional defects found while evaluating (not filed as issues)

These came up while checking the issues above. None has its own issue; several deserve one.

| Finding | Where | Verified | Suggested priority |
|---|---|---|---|
| `debugfs rdump` of a fast symlink overflows a heap buffer (the subject of open PR #298) | `debugfs/dump.c` | Reproduced under ASAN on `maint`; plain `maint` and `master` binaries silently create a wrong link target | High — merge PR #298 (correct, includes a test, applies to both branches) |
| `e2fsck -c` and `mke2fs -c` `sprintf()` the badblocks command into a 1024-byte stack buffer; a device path over about 1000 bytes crashes both. The same string goes unquoted to the shell through `popen()` | `e2fsck/badblocks.c`, `misc/mke2fs.c:test_disk()` | Crash reproduced by two evaluators on both branches; word-splitting reproduced; one evaluator also reproduced shell interpretation of the name | Medium — reachable only through the invoking user's own argument, but these tools normally run as root. PR #239 covers only the mke2fs buffer |
| `copy_fs_verity_data()` still takes a 32-bit `ext2_off_t` file offset; for an fs-verity source file ≥ 2 GiB the Merkle-tree offset is truncated, and ≥ 4 GiB it can wrap and overwrite file data with exit 0. The `6359e0ec` commit message's reason for leaving it is mistaken | `misc/create_inode.c:587` | Read (needs an fs-verity source file to run). I confirmed `ext2_off_t` is `__s32` and the signature | Medium — one-line fix |
| `debugfs rm`/`rmdir` never free the external xattr block or EA inodes | `debugfs/debugfs.c` | Re-run | Medium — see #97 |
| `debugfs` raw xattr dump over-read (PR #291) and htree dump over-read (PR #290) | `debugfs/xattrs.c`, `debugfs/htree.c` | Reproduced under ASAN on `maint` | Medium — both PRs are right but lack tests; #290 leaves a gap around unvalidated `info_length` (reasoned, not demonstrated) |
| `e2fsck -fn` exits 0 on a file system larger than its device | `e2fsck/unix.c` | Reproduced | Low–Medium; needs a decision |
| `tune2fs` honours only the last `-E`; an over-long `mount_opts` prints an error but exits 0 | `misc/tune2fs.c` | Reproduced | With #302 |
| `dumpe2fs` rejects `image?offset=N` and its "Unable to resolve" message prints `argv[1]` rather than the device | `misc/dumpe2fs.c:677-680` | Reproduced | Low |
| `debugfs ea_set -f` silently truncates its input to one block | `debugfs/xattrs.c:302-305` | Observed by one evaluator | Low |
| Several test scripts use `exit 0` although they are sourced (`m_assume_storage_prezeroed`, `f_detect_junk`, `m_roottar`, `m_rootgnutar`, `m_rootpaxtar`, `u_direct_io`) | `tests/*/script` | Read | Low |
| `./configure --enable-maintainer-mode && make -C po all` fails with a gettext infrastructure mismatch on `maint` | `po/Makefile.in.in` | Reproduced | Low; normal builds unaffected |

**Resolved on `github/maint` and `github/master` since 2026-11-26:**
- `badblocks -w` (`misc/badblocks.c:test_rw()`): corruption missed after a short read — fixed by `b2caa055` (`b_rw_mixed_faults`); false corruption reports with `-t random` or 3-byte patterns after any I/O error — fixed by `9c99ce5e` (`b_rw_random_err`). The same `maint` series also fixed a heap buffer overflow in `test_nd()` after short reads (`42a73b63`), 32-bit block-number wraparound near `2^32 - 1` (`13889d17`), block 0 end-of-list handling (`bf7996b9`), batch-size recovery after known-bad clamping (`44850e62`), and `test_nd()` multi-pattern state (`26ff7213`).

## Open pull requests read in the course of this work

37 open pull requests remain on GitHub (down from 45 after closing 8 superseded PRs on 2026-11-27, archived below). Only PRs that bear on an issue or sit in code an issue touches were assessed.

| PR | Subject | Assessment |
|---|---|---|
| #304 | fgetflags uninitialized | Merge with `lib/e2p/getflags.c` added (cleaner style than #281; fixes #271) |
| #303 | fuzzer `open()` mode | Merge (fixes #286) |
| #298 | rdump fast-symlink overflow | Merge; real heap write overflow, good test |
| #297 | fuse2fs inherited image locks | Do not merge as is; `master`-only code, needs a design decision |
| #296 | fuse2fs norecovery read-only | Merge with changes; applies to `master` only, fuse4fs not updated |
| #291 | debugfs raw xattr bounds | Merge with a test |
| #290 | debugfs htree count/limit | Merge with changes and a test |
| #288 | `ext2fs_xattr_inode_max_size` bounds | Not yet fixed on `maint` or `master` (diff read, not tested) |
| #285 | bound inline data size | Merge with changes (see #282) |
| #284 | fuse2fs missing `stdbool.h` | Merge to `master`/`next` only; not applicable to `maint` |
| #281 | fgetflags uninitialized | Close in favour of #304 (or merge with style changes and `getflags.c`; #271) |
| #280 | tolerate unsupported xattrs in `mke2fs -d` | Merge with a small change (`errno` is still tested after `ext2fs_free_mem()`) |
| #279 | GCC version check in tdb.c | Merge; the old test is false for any X.0 compiler |
| #276 | parallel library build ordering | Do not merge as is; not reproduced in 8 clean `-j56` builds, and it makes every no-op `make` re-archive five libraries |
| #267 | autopoint with gettext 1.0 | Do not merge alone; with the shipped `configure` it installs 0 `.mo` files instead of 24 |
| #266 | debugfs `rdump_tar` | Feature; merge with changes after #298 (it copies the same overflow; test uses `exit 0`) |
| #263 | selinux xattrs from tar | Merge with changes (#255) |
| #262 | e4defrag best-count option | Optional; does not address #300/#155 |
| #252 | libmagic optional | Right approach, three defects (#133) |
| #239 | mke2fs `test_disk` overflow | Real bug, incomplete patch; e2fsck has the identical code |
| #237, #121 | badblocks speed output | Not mergeable as is; #237 shows wrong speed with a first block and hand-edits 25 `po/` files |
| #236 | badblocks non-zero exit on errors | Merge with changes, coordinated with the #111 fix |
| #154 | badblocks status output | Fixes a real `-vv` garbling but only for that mode; needs rework |
| #135 | debugfs `rdump -p` | Does not apply; patches generated `configure` only; resets the process umask; needs a rewrite |
| #132 | filefrag: no filename for one file | Changes default output; ask for opt-in or close (#88) |
| #126 | uid/gid of copied inodes with `-d` | Do not merge as is; hunks fail, environment-variable interface, no validation |
| #123 | libunwind backtraces | Close; dead where `backtrace()` is absent, superseded by #104's fix |
| #15, #9 | badblocks/e2fsck options from 2017–18 | Stale; do not apply |

**Not assessed:** PRs #294, #275 (touched `badblocks.c` for DIO alignment / xorshift64 / SIGINT error counts + a duplicate CI workflow; conflicts with the new `badblocks` refactor), #268 (diff read only in passing), #243 (`mklost+found.c` `return` vs `exit`), #167, #161 (noted only for its licensing relevance), #116.

## Closed since initial triage

### Closed on 2026-11-27 (2 issues, 8 PRs — fixed/superseded on `github/maint` and `github/master`)

All 2 issues and 8 PRs below were closed on 2026-11-27 (verified against GitHub). Every cited commit is in both `github/master` (`76b5934a`) and `github/maint` (`cf354a9d`) and will ship in the next release after v1.47.4.

#### Issues closed (2)

| # | Title | Fixing commit(s) | Target release | Residual notes / optional follow-ups |
|---|---|---|---|---|
| #219 | badblocks: wrong CORRUPTION_ERROR block numbers | `b2caa05541ec` badblocks: fix corruption reporting and short-read check in `test_rw()` (test `b_rw_mixed_faults`); companion `9c99ce5eb3c2` (`b_rw_random_err`) | next release (in `maint` & `master`) | Superseded PR #142 (also closed). |
| #259 | ext2fs_write_new_inode: large inode extra timestamp not written for inode size 256B | `69963f8cc688` libext2fs: fix handling of inode timestamps before 1970 and after 2038, with companions `2aa2abb77901`, `a53a9723f574`, `841b2d36aad1`, `812d03293b71`, `3c1e51b5d776`, `5b3a591673aa` | next release (in `maint` & `master`) | Superseded PR #301 (also closed). Release-note item: `ext2fs_inode_xtime_get()`/`_set()` are header macros, so external applications pick up the fix upon rebuild. |

#### Pull requests closed as superseded (8)

| PR | Subject | Superseding commit(s) | Residual notes |
|---|---|---|---|
| #141 | badblocks: Skip known bad blocks correctly in non-destructive mode and write mode | `44850e62ed8c`, `bf7996b9b78e`, `bb75f4e9d40f`, `26ff7213df91` (4 `b_*` tests) | Covers the `badblocks -i` half of #189; `e2fsck -c -k` passing `-i` remains open in #189. |
| #142 | badblocks: Fix wrong corruption block record in write mode | `b2caa05541ec` (`b_rw_mixed_faults`), companion `9c99ce5eb3c2` (`b_rw_random_err`) | Closed alongside #219. |
| #270 | `ext2fs_add_journal_device` buffer overflow | `725a3bbfb943` (`tst_mkjournal`) | Checks `nr_users >= JBD2_USERS_MAX` after duplicate-UUID check and returns `EXT2_ET_TOO_MANY_REFS`. |
| #274 | e2fsck: validate fast commit dentry tag length | `69704f6d3aee` | Validates dentry tag length, inodes, and names in scan and replay via `fc_validate_dentry_tag()`. |
| #283 | libext2fs: invalidate cached blocks after a zeroout | `cb26ee7bce87` (`tst_unix_io`) | Covers both `unix_zeroout()` and `unix_discard()`; PR #283's `tests/d_expand_dir_zeroout` could optionally be cherry-picked. |
| #287 | bound inline symlink buffer in `follow_link` | `7a707c07aa9f` (`tst_dir`) | Validates both slow and inline symlinks. |
| #289 | e2fsck: bound fast commit replay tags to the journal block | `ac352aa79fd5` (`j_fc_scan_overread`) + fast-commit series through `2080baa6bd3b` (`69704f6d3aee`, `f5b72def6177`, `668a7e66dade`) | Validates tag headers, tag lengths, and per-tag minimum payload sizes in scan and replay. |
| #301 | libext2fs: fix large inode timestamp encoding | `69963f8cc688` + 6 companions (`2aa2abb77901`..`5b3a591673aa`) | Closed alongside #259. |

### Closed on 2026-11-26 (14 issues — already fixed in released versions)

All 14 issues below were closed on 2026-11-26 (verified against GitHub). Every fixing commit is in `github/master` and `github/maint`.

| # | Title | Fixing commit | First release | Residual notes / optional follow-ups |
|---|---|---|---|---|
| #56 | e4defrag crash with inline_data | `fdabe7cb` e4defrag: don't try to defragment files which have 0 or 1 blocks | v1.47.4 | Optional: add defensive guards in `e4defrag.c` (`orig_group_head == NULL`, `block_count == 0`, skip `FIEMAP_EXTENT_DATA_INLINE`) for an unverified inline-data + external-xattr-block corner case. |
| #81 | m_rootdir_acl test depends on CONFIG_EXT4_FS_POSIX_ACL=y | `13f1ce96` tests: fix ACL-printing tests | v1.46.6 | — |
| #85 | Can't repair ext4 partition after power outage, e2fsck keeps crashing | `54183fea` e2fsck: skip sorting extents if there are no valid extents | v1.46.5 | `668a7e66` (now in `maint` and `master`) also rewrote the function; both prevent the underflow. |
| #89 | debugfs: Cannot set the last write time to zero | `b6e29130` libext2fs: add support for the SOURCE_DATE_EPOCH environment variable | v1.47.1 | Optional: two-line fix in `do_set_current_time()` so interactive `set_current_time 0` also works; documenting `SOURCE_DATE_EPOCH` is tracked in #91. |
| #92 | RFE: mke2fs: allow setting filesystem metadata externally | `7e3a4f0a` mke2fs: the -d option can now handle tarball input | v1.47.1 | Remaining xattr gap for tar input is tracked in #255. |
| #96 | m_rootdir_acl test faild on 1.46.4 when selinux enabled | `13f1ce96` tests: fix ACL-printing tests | v1.46.6 | See #199 for plausible remaining XFS root fragility (`trusted.SGI_ACL_*`). |
| #120 | Tune2fs and fuse2fs do not change j_tail_sequence in journal superblock | `003125b2` tune2fs: do not change j_tail_sequence in journal superblock | v1.46.6 | Commit subject says the opposite of what the patch does. |
| #158 | m_rootdir_acl test fails when run from a btrfs filesystem | `487d8fb8` Accept alternate ordering of ACLs from btrfs | v1.47.3 | See #199 for plausible remaining XFS root fragility. |
| #173 | e2scrub finds corruption immediately after mounting | `4ee590f5` e2scrub: skip filesystems that don't have journals | v1.47.1 | Optional: mention the journal requirement in `e2scrub.8` and `e2scrub_all.8`. |
| #178 | Building on cygwin | `72e30620` libext2fs: fix ext2fs_get_device_size2() return value on Windows | v1.47.1 | Removed the `windows_io_manager` error; full Cygwin build unverified. |
| #235 | fuse2fs incorrectly assigns to stdout/stderr | `f79abd85` fuse2fs: fix logging redirection | v1.47.4 | — |
| #240 | Error when building debugfs against Kernel 5.10 headers | `4aba12dc` configure: check for FS_IOC_READ_VERITY_METADATA availability | v1.47.4 | — |
| #254 | Wrong (or at least confusing) "Ext2 file too big" message from `mkfs.ext4` | `6359e0ec` misc/create_inode.c: Fix for file larger than 2GB | v1.47.4 | Duplicate of #264; see next row. |
| #264 | overflow error with mke2fs | `6359e0ec` misc/create_inode.c: Fix for file larger than 2GB | v1.47.4 | Follow-ups: add a regression test covering offsets ≥ 2 GiB and ≥ 4 GiB; note in release notes that 1.47.3 could silently misplace data in [4 GiB, 6 GiB) with exit 0; fix the same 32-bit `ext2_off_t` still in `copy_fs_verity_data()` (see "Additional defects"). |

## Limitations and things I got wrong along the way

- **#282 was not executed.** It is rated High on code reading alone. The arithmetic is straightforward, I confirmed the two cited sites myself, and the evaluator searched pass 1 and the xattr reader for a guard and found none; but the exact failure and e2fsck's behaviour with PR #285 applied are unconfirmed.
- **#145/#108 images were built with libext2fs, not a kernel.** A loop-mounted `fallocate -n` test would confirm kernel-written files hit it.
- **Scratch patches are evidence, not fixes.** For #295, #292, #174 and #98 evaluators tried the sketched change in copies under `/tmp` to check that the diagnosis was right. None was reviewed, and the #174 one in particular is thinly tested.
- **My initial brief to the evaluators wrongly said `pu` was simply ahead of `master`.** One evaluator caught it; I confirmed the divergence, corrected the brief, and every group then checked both branches. Two evaluators declined to act on my correction because of how it was delivered, but both established the topology independently with `git` and did the two-branch comparison anyway.
- **Code search:** the `cs` tool does not index this local Git checkout, so `git grep` and `git log -S` were used throughout.
- **Priorities are my judgement.** Where I departed from an evaluator's rating (#219, initially rated High → Medium before it was fixed and closed on 2026-11-27) the original finding records why.

## Detailed findings

The per-area findings files hold the full evidence from the 2026-11-22 evaluation (including the 16 issues closed on 2026-11-26 and 2026-11-27): commands run, output, file and line citations, and longer design notes.

- [A-memsafety.md](file:///usr/local/google/home/tytso/.gemini/jetski/brain/d5b26b05-0fd2-41d7-8c1c-e345b2f7a6dd/scratch/findings/A-memsafety.md) — #282, #295, #259, #271, #286, #228, #227, #188, #198
- [B-e2fsck.md](file:///usr/local/google/home/tytso/.gemini/jetski/brain/d5b26b05-0fd2-41d7-8c1c-e345b2f7a6dd/scratch/findings/B-e2fsck.md) — #85, #108, #145, #139, #119, #115, #120, #78, #59, #173
- [C1-resize.md](file:///usr/local/google/home/tytso/.gemini/jetski/brain/d5b26b05-0fd2-41d7-8c1c-e345b2f7a6dd/scratch/findings/C1-resize.md) — #292, #174, #117, #98, #191, #20
- [C2-mke2fs.md](file:///usr/local/google/home/tytso/.gemini/jetski/brain/d5b26b05-0fd2-41d7-8c1c-e345b2f7a6dd/scratch/findings/C2-mke2fs.md) — #264, #254, #249, #95, #238, #91, #92, #76, #255, #163
- [D-badblocks.md](file:///usr/local/google/home/tytso/.gemini/jetski/brain/d5b26b05-0fd2-41d7-8c1c-e345b2f7a6dd/scratch/findings/D-badblocks.md) — #219, #211, #207, #190, #189, #175, #113, #111, #77
- [E1-build.md](file:///usr/local/google/home/tytso/.gemini/jetski/brain/d5b26b05-0fd2-41d7-8c1c-e345b2f7a6dd/scratch/findings/E1-build.md) — #240, #217, #179, #178, #176, #133, #127, #104, #38, #16
- [E2-pkg-tests.md](file:///usr/local/google/home/tytso/.gemini/jetski/brain/d5b26b05-0fd2-41d7-8c1c-e345b2f7a6dd/scratch/findings/E2-pkg-tests.md) — #199, #33, #32, #158, #134, #96, #81, #299, #86
- [F1-tune2fs-debugfs.md](file:///usr/local/google/home/tytso/.gemini/jetski/brain/d5b26b05-0fd2-41d7-8c1c-e345b2f7a6dd/scratch/findings/F1-tune2fs-debugfs.md) — #302, #195, #100, #99, #37, #130, #90, #89, #79, #63, #156
- [F2-fuse-defrag-misc.md](file:///usr/local/google/home/tytso/.gemini/jetski/brain/d5b26b05-0fd2-41d7-8c1c-e345b2f7a6dd/scratch/findings/F2-fuse-defrag-misc.md) — #242, #235, #138, #300, #155, #137, #56, #136, #88, #97, #93, #30

Reproducer scripts, images and logs are under `/tmp/e2fs-issue-eval/` (one directory per area, plus out-of-tree builds of `pu` and `master`). That directory is scratch and can be deleted; it holds a few large sparse images.
