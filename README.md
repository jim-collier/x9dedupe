# x9alldedup<!-- omit in TOC -->

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

## Footnotes

¹ Features provided through [rmlint](https://github.com/sahib/rmlint).

## To-do

- [ ] Test with latest master version of [rmlint](https://github.com/sahib/rmlint).
- [ ] Start C# version with no dependencies other than sqlite3.
  - Because I'm significantly more fluent in C# than C++, and with dotnet core 3.1, it has come a long way.
    - C# is still not perfect in terms of producing a single small exe with no dependencies. Either the dotnet runtime has to already be installed, or it's bundled with every exe and is extracted before running.
    - Another problem: the default sqlite3 ADO provider is currently broken when compiling a single binary, due to conflicting paths. This will likely be fixed at some point in the future. For now, using a single-file C# wrapper (sqlite-net).
- ~~Start C++ version with no external dependencies (other than statically linked libraries like sqlite3).~~
  - After a start, decided against C++. Too much cognitive overhead. For pure static binaries, Rust would be a better choice (or better yet, Go).
