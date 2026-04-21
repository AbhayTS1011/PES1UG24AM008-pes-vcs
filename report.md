Q5.1 — Implementing pes checkout <branch>
        To checkout a branch, three things must change in .pes/:

        .pes/HEAD must be updated to ref: refs/heads/<branch>
        Working directory must be updated — read the target branch's commit, walk its tree, and write each blob to the corresponding file path
        .pes/index must be updated to reflect the new tree's contents

        What makes it complex: you must handle files that exist in the current branch but not the target (delete them), files that exist in both (overwrite), and files only in the    target (create them).

Q5.2 — Detecting dirty working directory conflicts
        Compare three things for each tracked file:

        Hash in the current index
        Hash in the target branch's tree
        Hash of the actual file on disk

        If the disk file differs from the index (unstaged changes) AND the target branch has a different version of that file, refuse checkout. You detect this without re-hashing by using the mtime/size metadata in the index for a fast check.

Q5.3 — Detached HEAD
        In detached HEAD state, new commits are created but no branch pointer is updated — only HEAD itself points to the new commit. If you switch branches, those commits become unreachable since no branch references them. To recover, you can create a new branch pointing to the detached commit hash: git branch recovery <hash>.

Q6.1 — Garbage Collection Algorithm
        Use a mark-and-sweep approach:

        Start from all branch refs in .pes/refs/heads/
        Walk every commit chain following parent pointers
        For each commit, mark its tree and recursively mark all blobs and subtrees
        Use a hash set to track all reachable object IDs
        Scan all files in .pes/objects/ and delete any not in the hash set

        For 100,000 commits and 50 branches you would visit roughly 300,000–500,000 objects (commits + trees + blobs), assuming average 3–5 objects per commit.

Q6.2 — GC Race Condition
        Race condition scenario:

        A commit operation writes a new blob object to the store
        GC scans the object store and doesn't see the blob referenced by any commit yet (commit not written yet)
        GC deletes the blob as unreachable
        The commit operation finishes writing the commit object — but the blob it references is now gone, causing corruption

        Git avoids this by using a grace period — GC never deletes objects newer than 2 weeks old, giving in-progress operations time to complete and reference them.
