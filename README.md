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

## Footnotes

¹ Features provided through [rmlint](https://github.com/sahib/rmlint).

## To-do

- [ ] Test with latest master version of [rmlint](https://github.com/sahib/rmlint).
- [ ] For status matascript generation: Use heredoc format rather than per-line echo/redirect.
- [ ] Put in latest template wrapper, mainly for 'cheap' access to profiling.
- [ ] Rename stuff like local -r funcName="$(basename "$0").fMain()" to "${meName}.${FUNCNAME[0]}"
- [ ] Create a status file in /tmp that calling scripts can check (last instance of). See TODOs in script for details.
- [ ] Catch 'exit' event and do housecleaning. See TODOs in script for details.
- [ ] Throw error if spaces in rmlint filespec arguments.
  - [ ] Also, double-check to see if this is really necessary, refresh why, and how to fix it rather than working around it.
- [ ] Get opening realtime output in another terminal window working. (But really? Feels like an antipattern. This needs to be able to run CLI-only.)
- [ ] Show near-realtime filesystem space in status script.
- [ ] Start a compiled, cross-platform version of this with no dependencies other than sqlite3.
  - Not too difficult compared to `rmlint` (which covers a huge and unwieldly problem domain).
  - This script already covers most of the logic [when deduping ZFS] that has to be done, and what's not covered, is well-defined:
    - Determining what files have the same content (CRC64 then blake2b).
      - How to avoid whole-file scans (first compare mtime and size, then first and last bytes).
    - Determining what files don't need to be compared again.
    - Determining what files have already been deduped.
    - How to both increase performance, reduce work, _and_ be resilient to losing local cache.
      - Maintain a sqlite3 database with file attributes.
      - Store in per-file Xattrs, minimally due to limited space: checksum (CRC64 or blake2b but not both), last mtime, last size, last inode, last checked, last scanned, last deduped.
        - Reuse rmlint's attributes where appropriate, to save xattr space and time.
  - Language options:
    - C#
      - Pros:
        - I'm (JC) know C# reasonably well, with a decent well-tested toolkit.
        - The open-source Dotnet Core (currently v3.1 and C# v8) has made impressive progress in a short amount of time.
        - Can generate single-file platform-specific executables, from any platform.
      - Cons:
        - Fine-grained native filesystem handling is woefully lacking. Would almost certainly require either complex native interops with platform preprocessor directives, or an interop wrapper around a custom C/C++ library, that in turn wraps the same things with a possibly vastly simpler interface for C# to work with.
        - C# has a long way to go (if ever?) to produce a single small exe with no dependencies. Either:
          1. The correct dotnet runtime has to already be installed, or
          1. The executable is bundled with every exe and is extracted before running.
        - There's no way to bundle or statically link the sqlite3 library into the executable. (As can be done with most other C-like compiled languages.) The correct version has to already be installed. (It's possible to ship per-platform binaries with the compiled project executable, but it's dizzingly complex to pull off.)
        - The default sqlite3 ADO provider is currently broken when compiling a single binary, due to conflicting paths. This will likely be fixed at some point in the future. For now, using a single-file C# wrapper (sqlite-net).
          - Workaround: Don't use sqlite3. Use the dotnet Data[Table|Set|View] classes. But for large filesystems this could exhaust memory. (Similar to the ZFS inline deduplication problem.) But there are workarounds to that as well, if you're willing to trade lower memory for higher CPU horsepower and slower performance [that grows O(log n) or even exponentially with file count]. For example:
            1. Rather than storing every file as a fully qualified filesystem path, break them up into a self-relational graph model to significantly reduce redundancy.
            1. Rather than storing a blake2b checksum for every file, only store a CRC64, and fall back to blake2b in another table only for collisions.
      - Summary: C# is a really nice, safe, fast language to work with. But while a distributable executable will easily run on a variety of platforms, that advantage evaporates when you consider that, at minimum, either the exe is going to be huge or the correct .net runtime must be installed; and either way the expected version of sqlite3 can't realistically be statically linked, and must also be included.
    - C++
      - Pros:
        - Fastest, smallest binaries, wide developer support.
        - Relatively easy to compile and statically link a fixed version of sqlite3 into the project.
        - C++ 17 has come a long way with memory safety.
        - Easiest way to maintain a project over many years, even decades. (With all dependencies in source tree.)
      - Cons:
        - I (JC) don't know it very well, esp. C++ 17.
        - Arcane & often anacronistic syntax.
        - Getting tooling up and running is always a major headache, especially for cross-platform projects, and is often brittle.
    - Go
      - Pros:
        - Fast, small native binaries.
        - Strongly opinionated, which is great for learning a new language. (Opposite of Rust or especially C++.)
        - Can statically link in sqlite3.
      - Cons:
      	- I'm (JC) used to OO. Learning curve.
      	- Current Go programmers admit it still has many shortcomings (many of which are currently being worked on), as a language, compared to C#.
