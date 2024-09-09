# x9dedupe<!-- omit in TOC -->

## Table of contents<!-- omit in TOC -->

- [Overview](#overview)
- [Footnotes](#footnotes)
- [To-do](#to-do)

## Overview

The first and so far only ZFS offline deduplication tool. Can also efficiently deduplicate Btrfs and XFS.

Currently relies partially on [rmlint](https://github.com/sahib/rmlint), though a planned binary version will remove external dependencies.

Deduplicates the specified directory and below, via one of three explicit options. Duplicate files aren't deleted, but their redundant space is recovered. Includes the only known high-performance 'offline ZFS deduper' (as of 2019-Aug). All dedup methods share these common features:

- Deduplicates redundant file content, even if the files have different names, dates, etc. File names and locations don't matter and can frequently change between runs, without affecting speed, safety, or space-savings.¹
- Fast file compare; only hashes files if other attributes fail to rule out non-dupes such as filesize, first & last few bytes, and pre-computed hash.¹
- Stores extended file attributes to both incrementally compare, and incrementally dedup. Significantly speeds up successive runs and reduces disk wear.¹
- Xattrs can survive copying to different filesystems & across networks, with optional flags built in to most modern Linux copy utilities (eg cp, rsync).
- Safe; data is not lost even in the case of error or crash.
- Blake2b 512-bit hash algorithm is extremely fast and robust¹ but even in the case of nearly impossible collisions, neither Btrfs nor ZFS will clone extents or blocks, unless bit-for-bit identical.
- Checks to make sure files aren't in use before cloning.
- Simplifies and extends 'rmlint', tested as the fastest, most robust, and well-maintained dedup utility. Git page: https://github.com/sahib/rmlint/
- Requires a recent version of rmlint, from v2.8.0 master branch, v2.9.0 release branch, or higher. v2.9.0 was released on 2019-08-20.

On the negative side, ZFS requires more memory even just to *read* deduplicated data, even after live deduplication is turned off.

## Footnotes

¹ Features provided through [rmlint](https://github.com/sahib/rmlint).

## To-do

- [ ] Provide a warning that deduplicated data incurs large ZFS memory requirement, even after deduplication is done.
- [ ] Test with latest master version of [`rmlint`](https://github.com/sahib/rmlint) (use fclones instead).
    - [ ] Consider using [fclones](https://github.com/pkolaczk/fclones) instead. A newer, simpler utility written in rust.
- [ ] Rather than having `rmlint` generate a script, have it output a list of files to process.
- [ ] Put in latest template wrapper, mainly for 'cheap' access to profiling.
- [ ] Rename stuff like local -r funcName="$(basename "$0").fMain()" to "${meName}.${FUNCNAME[0]}"
- [ ] Create a status file in /tmp that calling scripts can check (last instance of). See TODOs in script for details.
- [ ] Catch 'exit' event and do housecleaning. See TODOs in script for details.
- [ ] Throw error if there are spaces in rmlint filespec arguments.
  - [ ] Also, double-check to see if this is really necessary, refresh why, and how to fix it rather than working around it.
- [ ] Get opening realtime output in another terminal window working. (But really? Feels like an antipattern. This needs to be able to run CLI-only.)
- [ ] Show near-realtime filesystem space in status script.
- [ ] Start a compiled version of this with no dependencies other than sqlite3, written in Go.
  - Not too difficult compared to `rmlint` (which covers a huge and unwieldly problem domain).
  - This script already covers most of the logic [when deduping ZFS] that has to be done, and what's not covered, is well-defined:
    - Determining what files have the same content (CRC64 then blake2b).
      - How to avoid whole-file scans (first compare mtime and size, then first and last bytes).
    - Determining what files don't need to be compared again.
    - Determining what files have already been deduped.
    - How to both increase performance, reduce work, _and_ be resilient to losing local cache:
      - Maintain a sqlite3 database with file attributes.
      - Store in per-file Xattrs, minimally due to limited space: checksum (CRC64 or blake2b but not both), last mtime, last size, last inode, last checked, last scanned, last deduped.
        - Reuse rmlint's attributes where appropriate, to save xattr space and time.
