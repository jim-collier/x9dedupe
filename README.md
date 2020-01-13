# alldedup
The first and so far only ZFS offline deduplication tool. Can also efficiently deduplicate Btrfs and XFS.

Currently relies on [rmlint](https://github.com/sahib/rmlint), though a planned binary version will remove external dependencies.

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

Notes:
¹ Features provided through [rmlint](https://github.com/sahib/rmlint).
