# PES-VCS Lab Report

## Phase 5: Branching Analysis

### Q5.1
To implement `pes checkout <branch>`:
Update `.pes/HEAD` to write `ref: refs/heads/<branch>`. Read the target
branch's commit hash, read that commit's tree, and restore all files from
the tree objects into the working directory. You must traverse the entire
tree recursively, create/delete files and directories, and handle file
permission changes.

### Q5.2
To detect a dirty working directory conflict:
For each file in the index, compute its current on-disk hash and compare
to the index entry's stored hash. If different, the file is locally
modified. Also compare the index entry's hash to the same file's hash in
the target branch's tree. If both differ, the checkout must refuse.
Only the index and object store are needed — no extra metadata required.

### Q5.3
In detached HEAD state, new commits are created but no branch file is
updated. HEAD stores the commit hash directly. After switching away,
those commits become unreachable. Recovery: manually inspect
`.pes/objects/` to find the hash, then create a new branch pointing to
it.

## Phase 6: Garbage Collection Analysis

### Q6.1
Algorithm:
1. Start from all branch refs in `.pes/refs/heads/`.
2. Walk each commit chain following parent pointers, collecting hashes.
3. For each commit, collect its tree hash and walk each tree recursively,
   collecting all tree and blob hashes.
4. All collected hashes = reachable set.
5. Walk all files in `.pes/objects/` and delete any not in reachable set.

Data structure: a hash set of ObjectIDs for O(1) lookup.
For 100,000 commits across 50 branches with ~5 files each: roughly
700,000 objects visited minimum, 200,000-400,000 after deduplication.

### Q6.2
Race condition: GC identifies an object as unreachable and marks it for
deletion. Concurrently, `pes add` writes a blob with the same hash.
GC deletes it before `pes commit` writes the tree/commit referencing it.
Now the commit references a deleted blob.

Git avoids this via: (1) a grace period — never delete objects newer
than 2 weeks. (2) Lock files — GC only runs when no concurrent writes
are in progress. The temp-file-then-rename pattern ensures partially
written objects are never visible to GC.
