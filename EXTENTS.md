# Extents and defragmentation

This fork adds a third file-data representation to littlefs: **extents**
(immutable, contiguous runs of blocks) and a **defragmenter** that makes room
for them. It is the filesystem half of MicroPythonOS's littlefs2e: the
operating system uses it to run native code straight out of flash, which
requires the code to sit in one 64 KiB-aligned run instead of the scattered
blocks a normal littlefs write produces.

Everything else is littlefs v2.11.3 unchanged.

## What an extent is

An extent file is a normal file in the metadata tree whose data tag is an
extent descriptor instead of CTZ or inline data:

    struct lfs_ext {
        uint32_t head;    // first block of the run
        uint32_t count;   // padded run length in blocks
        uint32_t length;  // payload bytes
        uint32_t crc;     // optional payload CRC
        uint32_t flags;
    };

The tag is `LFS_TYPE_EXTSTRUCT` (`0x203`). The run is contiguous and
immutable: it is written once through `lfs_ext_write()`, then frozen by
`lfs_ext_commit()`. Opening an extent for writing or truncating it converts
the file back to ordinary littlefs data, and the run's blocks become free.

A run is chosen by `lfs_ext_reserve()`, which finds a window of free blocks
(optionally aligned), erases it and keeps it out of the allocator until
commit or abort. That reservation is in memory only; the metadata entry that
claims the blocks is written at commit, so a power loss before commit leaves
the blocks free and the file untouched.

## On-disk compatibility

The disk version stays **2.1**. There is no new on-disk version to mount and
no legacy version to convert.

Stock littlefs does not know `LFS_TYPE_EXTSTRUCT`, so:

- its traversal does not count an extent's run as used, and its allocator may
  hand those blocks out to another file;
- a stock `lfs_stat()` finds no data tag where it expects one, so the file
  reads as 0 bytes.

**Treat an image that may contain extents as read-only for stock tools**
(littlefs-python, mklittlefs, an older firmware). Writing to such an image can
silently overwrite an extent. The stored CRC lets a consumer detect that (or
plain flash corruption) before using the payload.

## API

```c
// create
int lfs_ext_reserve(lfs_t *lfs, lfs_size_t length, lfs_block_t align,
        lfs_ext_t *out);
int lfs_ext_write(lfs_t *lfs, lfs_size_t offset, const void *buffer,
        lfs_size_t size);
int lfs_ext_commit(lfs_t *lfs, const char *path, uint32_t crc);
void lfs_ext_abort(lfs_t *lfs);

// inspect
int lfs_ext_info(lfs_t *lfs, const char *path, lfs_ext_t *out);
lfs_block_t lfs_ext_largest_run(lfs_t *lfs, lfs_block_t align);
int lfs_ext_block_maps(lfs_t *lfs, ...);       // who owns which blocks
int lfs_ext_file_blocks(lfs_t *lfs, const char *path, ...);
int lfs_ext_dir_pairs(lfs_t *lfs, ...);        // metadata pairs by tag walk

// relocate and defragment
int lfs_ext_relocate_dir(lfs_t *lfs, lfs_block_t pair[2]);
int lfs_ext_clear_window(lfs_t *lfs, lfs_block_t head, lfs_block_t count);
int lfs_ext_defrag(lfs_t *lfs, lfs_size_t length, lfs_block_t align,
        lfs_ext_t *out);

// resumable defragmentation
int lfs_ext_defrag_begin(lfs_t *lfs, lfs_size_t length, lfs_block_t align);
int lfs_ext_defrag_step(lfs_t *lfs, uint8_t *phase, uint8_t *percent);
int lfs_ext_defrag_result(lfs_t *lfs, lfs_ext_t *out);
void lfs_ext_defrag_abort(lfs_t *lfs);

// allocator window exclusion (used by the defragmenter)
int lfs_ext_exclude(lfs_t *lfs, lfs_block_t head, lfs_block_t count);
```

`reserve()` returns `LFS_ERR_NOCONTIG` when no run of that size (at that
alignment) exists — that is the signal to defragment.

## Defragmentation

`lfs_ext_defrag()` clears one window and reserves it as a run:

1. Rank aligned windows by how many used blocks they hold; prefer the emptiest.
2. Reject a window holding an extent, the root metadata pair, or a split
   directory's **tail** pair. Drop a window holding an open file's blocks and
   try the next candidate — a caller may legitimately hold a file open (an
   application reading its own asset) and one busy window must not abort the
   run.
3. `lfs_ext_clear_window()` excludes the window from the allocator, relocates
   every directory whose pair intersects it (`lfs_ext_relocate_dir()`), then
   copies each occupying ordinary file to `<name>.dfg` and renames it back.
   Occupants are processed in rounds, so a window with many files needs no
   large table; rounds are capped by the window size (`count / occupants + 2`),
   not a fixed number.
4. The cleared window is erased and reserved exactly as `lfs_ext_reserve()`
   does, so `lfs_ext_commit()` can turn it into an extent.

`lfs_ext_relocate_dir()` compacts a directory into a fresh metadata block,
repoints the parent (following the chain when the parent relocates too) and
unlinks the old pair: copy, switch, orphan, never move in place. A compaction
moves only one half of the pair (`{a,b}` becomes `{c,a}`), so the caller
repeats while either block is inside an active exclusion.

The window scan walks directories **by pair**, from `DIRSTRUCT` tags, not by
path: a path lookup costs a full metadata scan per path component, while one
`lfs_dir_get` per entry covers every data tag and the name is read only for
entries the window keeps.

The `begin`/`step`/`result`/`abort` machine exists so a caller that owns a UI
can run the same work incrementally, yielding between steps. Each step reports
a phase (`SPACE`, `MOVE`, `ERASE`) and a best-effort percentage. Only one run
is active at a time, and a mount discards a stale one (a non-local exit such
as a power cut would otherwise leave it behind).

### Limits

- **A split directory's tail pair cannot be relocated.** `lfs_fs_parent()`
  finds a child through its `DIRSTRUCT` entry, while a tail is referenced by a
  `TAIL` tag in the head pair. A window holding one is skipped, which is the
  one remaining ceiling on the largest obtainable run. Lifting it means
  rewriting the head's `TAIL` tag to point at a freshly allocated tail pair,
  or rebuilding the head and its whole tail chain into new pairs.
- **Defragmenting needs scratch space**: copying an occupant needs its size in
  free blocks, so a nearly full filesystem cannot always produce a large run.
- The cost is the device, not the algorithm. On a MicroPythonOS badge
  (7 MiB image, ~270 four-block files, 48-block window) a full clear is ~11 s:
  ~2 s scanning, ~3.4 s moving, ~2.1 s erasing (erase is ~43 ms per 4 KiB
  block and writes run at ~58 KB/s, so the moves are write-bound).

## Tests

- `tests/test_extent.toml` — create/read/stat/rename/remove, alignment 1..8,
  `O_TRUNC` converting an extent back to ordinary data, the disk version
  staying at 2.1 across a remount, and a clean `LFS_ERR_NOCONTIG`.
- `tests/test_extent_defrag.toml` — relocation contents and remount, root
  refusal, the two-pass window clear, the parent chain through injected bad
  blocks, a tail pair pinning a window, a fragmented image the defragmenter
  must clear and reserve, budget/extent refusals, a 7 MiB badge-sized image,
  the resumable driver, and a window of one-block occupants (round budget).
  Two cases are `reentrant`, so the runner replays them under power loss.

They share helpers in `relocate_test_helpers.h`, which the `Makefile` injects
with `--include`; the header includes `runners/test_runner.h` itself, so this
is safe for every case file. It is kept in both the repo root and `tests/`
because a quoted include is first resolved next to the including file
(`tests/*.t.c` and `bd/*.t.c`).

    make test NO_COV=1 -j8

Power loss is `-P none,linear` (one cut at every write offset) plus fixed
two-cut sequences where a relocation and a file move interleave:

    make test NO_COV=1 -j8 TESTFLAGS="tests/test_extent.toml \
        tests/test_extent_defrag.toml -P none,linear,{1,1},{1,2},{2,1},{3,3}"

**Do not use `-Pexhaustive`**: it forks a copy of the whole block device at
every cut point and keeps the branch tree, which needs gigabytes for a case
with a few dozen writes (it has OOM-killed a 7.7 GB machine).
