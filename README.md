# PES-VCS Lab Report
**Name:** Diya R Gowda  
**SRN:** PES1UG24CS159  

---

## Phase 1: Object Storage

**Screenshot 1A:** `./test_objects` output showing all tests passing  
<img width="810" height="223" alt="image" src="https://github.com/user-attachments/assets/48532615-d081-4049-a11e-b7f9f5cd80db" />

**Screenshot 1B:** `find .pes/objects -type f` showing sharded directory structure  
<img width="812" height="97" alt="image" src="https://github.com/user-attachments/assets/731e0261-4587-4c1f-bdbe-4afcfbb6b7ee" />


---

## Phase 2: Tree Objects

**Screenshot 2A:** `./test_tree` output showing all tests passing  
<img width="812" height="142" alt="image" src="https://github.com/user-attachments/assets/3ef94694-2e1b-4936-bd59-c1445fbd87e1" />

**Screenshot 2B:** `xxd` of a raw tree object (first 20 lines)  
<img width="829" height="67" alt="image" src="https://github.com/user-attachments/assets/874076c4-f701-43ea-934a-67ba15c3cdae" />


---

## Phase 3: The Index (Staging Area)

**Screenshot 3A:** `pes init` → `pes add` → `pes status` sequence  
<img width="804" height="561" alt="image" src="https://github.com/user-attachments/assets/8bd1025d-9dc6-47aa-9a80-0336d693734c" />

**Screenshot 3B:** `cat .pes/index` showing the text-format index  
<img width="811" height="80" alt="image" src="https://github.com/user-attachments/assets/f37e8a35-f059-4866-af5b-cd4c698192a7" />


---

## Phase 4: Commits and History

**Screenshot 4A:** `pes log` output showing three commits  
<img width="864" height="540" alt="image" src="https://github.com/user-attachments/assets/ce9b9f52-18bb-4950-aa5f-344362ae04b6" />

**Screenshot 4B:** `find .pes -type f | sort` showing object store growth  
<img width="860" height="380" alt="image" src="https://github.com/user-attachments/assets/20f7dc0f-864d-4c39-a9e1-78c9016aef04" />

**Screenshot 4C:** `cat .pes/refs/heads/main` and `cat .pes/HEAD`  
<img width="866" height="89" alt="image" src="https://github.com/user-attachments/assets/6efaef9a-89af-42b3-ada5-7da2ffa66424" />


---


**Final Screenshot:** `Final Integration Test`
<img width="909" height="595" alt="image" src="https://github.com/user-attachments/assets/8e9f376c-f3d9-45de-9ae2-c5245e26ab0e" />
<img width="911" height="719" alt="image" src="https://github.com/user-attachments/assets/2ecee119-94f4-4414-835a-13981bd19f43" />
<img width="906" height="476" alt="image" src="https://github.com/user-attachments/assets/3b3b9a71-2620-4a3e-8406-c249103da276" />

---

## Phase 5: Analysis — Branching and Checkout

**Q5.1: How would you implement `pes checkout <branch>`? What makes it complex?**

To implement `pes checkout <branch>`, two things need to happen. First, update `.pes/HEAD` to point to the new branch name (e.g., `ref: refs/heads/feature`). Second, read that branch's latest commit, get its tree object, and update every file in the working directory to match the tree's snapshot.

The complexity comes from handling the working directory safely. If the user has modified a tracked file that also differs between the two branches, blindly overwriting it would destroy their uncommitted work. So checkout must detect these conflicts and refuse to proceed, forcing the user to commit or stash their changes first. Additionally, files that exist in the current branch but not the target branch must be deleted, and new files in the target branch must be created — all atomically and safely.

---

**Q5.2: How would you detect a "dirty working directory" conflict using only the index and object store?**

Before switching branches, for each file tracked in the index, compute its current SHA-256 hash from the working directory and compare it against the hash stored in the index. If they differ, the file has been locally modified (it's "dirty"). Then also compare the index hash against the hash of the same file in the target branch's tree object. If the file differs in both places (working directory ≠ index AND index ≠ target tree), there is a genuine conflict and checkout should be refused. This approach avoids re-reading the full file content unnecessarily by using the index's stored mtime and size as a fast pre-check before hashing.

---

**Q5.3: What happens if you make commits in "detached HEAD" state? How do you recover?**

In detached HEAD state, HEAD contains a commit hash directly instead of a branch reference. Any new commits made in this state are stored correctly in the object store, but no branch pointer is updated to track them. This means once you switch away, those commits become unreachable — no branch points to them and they won't appear in `pes log`.

To recover, you need to find the hash of the last commit made in detached HEAD state (e.g., from terminal history or by inspecting `.pes/objects`), then create a new branch pointing to it by writing that hash into a new file under `.pes/refs/heads/` and updating HEAD to reference that branch.

---

## Phase 6: Analysis — Garbage Collection

**Q6.1: Describe an algorithm to find and delete unreachable objects. What data structure would you use?**

Starting from all branch refs in `.pes/refs/heads/`, perform a DFS or BFS traversal: for each branch, read its commit object, then recursively follow its parent pointer, and for each commit read its tree object and all blob objects it references. Mark every visited hash as "reachable" in a hash set (for O(1) lookup and insertion). After traversal is complete, scan all files under `.pes/objects/` and delete any whose hash is not in the reachable set.

For a repository with 100,000 commits and 50 branches, assuming roughly 10 tree and blob objects per commit on average, you'd visit approximately 1 million objects during the reachability traversal. A hash set implemented as a C hash table or a sorted array with binary search would handle this efficiently.

---

**Q6.2: Why is it dangerous to run garbage collection concurrently with a commit? Describe the race condition.**

Consider this sequence: a commit operation computes a new blob hash and is about to write it to `.pes/objects/`. At the same moment, GC scans the object store, does not find that hash yet (since the write hasn't happened), marks it as unreachable, and deletes it. When the commit then tries to write the commit object referencing that blob, the blob no longer exists — the repository is now corrupt.

Git avoids this race condition in two ways. First, it uses a "grace period" — any object file newer than 2 weeks is never deleted by GC, giving in-progress operations time to complete. Second, Git always writes all dependent objects (blobs and trees) fully to disk before writing the commit object that references them, and only updates branch refs last. This ensures that even if GC runs between steps, it will see the objects as too new to delete.

---

## Final Integration Test

**Screenshot:** `make test-integration` output showing all tests passing
