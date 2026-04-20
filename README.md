# Building PES-VCS — A Version Control System from Scratch

**Objective:** Build a local version control system that tracks file changes, stores snapshots efficiently, and supports commit history. Every component maps directly to operating system and filesystem concepts.

**Platform:** Ubuntu 22.04

---

## Getting Started

### Prerequisites

```bash
sudo apt update && sudo apt install -y gcc build-essential libssl-dev
```

### Using This Repository

This is a **template repository**. Do **not** fork it.

1. Click **"Use this template"** → **"Create a new repository"** on GitHub
2. Name your repository (e.g., `SRN-pes-vcs`) and set it to **public**. Replace `SRN` with your actual SRN, e.g., `PESXUG24CSYYY-pes-vcs`
3. Clone this repository to your local machine and do all your lab work inside this directory.
4.  **Important:** Remember to commit frequently as you progress. You are required to have a minimum of 5 detailed commits per phase. Refer to [Submission Requirements](#submission-requirements) for more details.
5. Clone your new repository and start working

The repository contains skeleton source files with `// TODO` markers where you need to write code. Functions marked `// PROVIDED` are complete — do not modify them.

### Building

```bash
make          # Build the pes binary
make all      # Build pes + test binaries
make clean    # Remove all build artifacts
```

### Author Configuration

PES-VCS reads the author name from the `PES_AUTHOR` environment variable:

```bash
export PES_AUTHOR="Your Name <PESXUG24CS042>"
```

If unset, it defaults to `"PES User <pes@localhost>"`.

### File Inventory

| File               | Role                                 | Your Task                                          |
| ------------------ | ------------------------------------ | -------------------------------------------------- |
| `pes.h`            | Core data structures and constants   | Do not modify                                      |
| `object.c`         | Content-addressable object store     | Implement `object_write`, `object_read`            |
| `tree.h`           | Tree object interface                | Do not modify                                      |
| `tree.c`           | Tree serialization and construction  | Implement `tree_from_index`                        |
| `index.h`          | Staging area interface               | Do not modify                                      |
| `index.c`          | Staging area (text-based index file) | Implement `index_load`, `index_save`, `index_add`  |
| `commit.h`         | Commit object interface              | Do not modify                                      |
| `commit.c`         | Commit creation and history          | Implement `commit_create`                          |
| `pes.c`            | CLI entry point and command dispatch | Do not modify                                      |
| `test_objects.c`   | Phase 1 test program                 | Do not modify                                      |
| `test_tree.c`      | Phase 2 test program                 | Do not modify                                      |
| `test_sequence.sh` | End-to-end integration test          | Do not modify                                      |
| `Makefile`         | Build system                         | Do not modify                                      |

---

## Understanding Git: What You're Building

Before writing code, understand how Git works under the hood. Git is a content-addressable filesystem with a few clever data structures on top. Everything in this lab is based on Git's real design.

### The Big Picture

When you run `git commit`, Git doesn't store "changes" or "diffs." It stores **complete snapshots** of your entire project. Git uses two tricks to make this efficient:

1. **Content-addressable storage:** Every file is stored by the SHA hash of its contents. Same content = same hash = stored only once.
2. **Tree structures:** Directories are stored as "tree" objects that point to file contents, so unchanged files are just pointers to existing data.

```
Your project at commit A:          Your project at commit B:
                                   (only README changed)

    root/                              root/
    ├── README.md  ─────┐              ├── README.md  ─────┐
    ├── src/            │              ├── src/            │
    │   └── main.c ─────┼─┐            │   └── main.c ─────┼─┐
    └── Makefile ───────┼─┼─┐          └── Makefile ───────┼─┼─┐
                        │ │ │                              │ │ │
                        ▼ ▼ ▼                              ▼ ▼ ▼
    Object Store:       ┌─────────────────────────────────────────────┐
                        │  a1b2c3 (README v1)    ← only this is new   │
                        │  d4e5f6 (README v2)                         │
                        │  789abc (main.c)       ← shared by both!    │
                        │  fedcba (Makefile)     ← shared by both!    │
                        └─────────────────────────────────────────────┘
```

### The Three Object Types

#### 1. Blob (Binary Large Object)

A blob is just file contents. No filename, no permissions — just the raw bytes.

```
blob 16\0Hello, World!\n
     ↑    ↑
     │    └── The actual file content
     └─────── Size in bytes
```

The blob is stored at a path determined by its SHA-256 hash. If two files have identical contents, they share one blob.

#### 2. Tree

A tree represents a directory. It's a list of entries, each pointing to a blob (file) or another tree (subdirectory).

```
100644 blob a1b2c3d4... README.md
100755 blob e5f6a7b8... build.sh        ← executable file
040000 tree 9c0d1e2f... src             ← subdirectory
       ↑    ↑           ↑
       │    │           └── name
       │    └── hash of the object
       └─────── mode (permissions + type)
```

Mode values:
- `100644` — regular file, not executable
- `100755` — regular file, executable
- `040000` — directory (tree)

#### 3. Commit

A commit ties everything together. It points to a tree (the project snapshot) and contains metadata.

```
tree 9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d
parent a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0
author Alice <alice@example.com> 1699900000
committer Alice <alice@example.com> 1699900000

Add new feature
```

The parent pointer creates a linked list of history:

```
    C3 ──────► C2 ──────► C1 ──────► (no parent)
    │          │          │
    ▼          ▼          ▼
  Tree3      Tree2      Tree1
```

### How Objects Connect

```
                    ┌─────────────────────────────────┐
                    │           COMMIT                │
                    │  tree: 7a3f...                  │
                    │  parent: 4b2e...                │
                    │  author: Alice                  │
                    │  message: "Add feature"         │
                    └─────────────┬───────────────────┘
                                  │
                                  ▼
                    ┌─────────────────────────────────┐
                    │         TREE (root)             │
                    │  100644 blob f1a2... README.md  │
                    │  040000 tree 8b3c... src        │
                    │  100644 blob 9d4e... Makefile   │
                    └──────┬──────────┬───────────────┘
                           │          │
              ┌────────────┘          └────────────┐
              ▼                                    ▼
┌─────────────────────────┐          ┌─────────────────────────┐
│      TREE (src)         │          │     BLOB (README.md)    │
│ 100644 blob a5f6 main.c │          │  # My Project           │
└───────────┬─────────────┘          └─────────────────────────┘
            ▼
       ┌────────┐
       │ BLOB   │
       │main.c  │
       └────────┘
```

### References and HEAD

References are files that map human-readable names to commit hashes:

```
.pes/
├── HEAD                    # "ref: refs/heads/main"
└── refs/
    └── heads/
        └── main            # Contains: a1b2c3d4e5f6...
```

**HEAD** points to a branch name. The branch file contains the latest commit hash. When you commit:

1. Git creates the new commit object (pointing to parent)
2. Updates the branch file to contain the new commit's hash
3. HEAD still points to the branch, so it "follows" automatically

```
Before commit:                    After commit:

HEAD ─► main ─► C2 ─► C1         HEAD ─► main ─► C3 ─► C2 ─► C1
```

### The Index (Staging Area)

The index is the "preparation area" for the next commit. It tracks which files are staged.

```
Working Directory          Index               Repository (HEAD)
─────────────────         ─────────           ─────────────────
README.md (modified) ──── pes add ──► README.md (staged)
src/main.c                            src/main.c          ──► Last commit's
Makefile                               Makefile                snapshot
```

The workflow:

1. `pes add file.txt` → computes blob hash, stores blob, updates index
2. `pes commit -m "msg"` → builds tree from index, creates commit, updates branch ref

### Content-Addressable Storage

Objects are named by their content's hash:

```python
# Pseudocode
def store_object(content):
    hash = sha256(content)
    path = f".pes/objects/{hash[0:2]}/{hash[2:]}"
    write_file(path, content)
    return hash
```

This gives us:
- **Deduplication:** Identical files stored once
- **Integrity:** Hash verifies data isn't corrupted
- **Immutability:** Changing content = different hash = different object

Objects are sharded by the first two hex characters to avoid huge directories:

```
.pes/objects/
├── 2f/
│   └── 8a3b5c7d9e...
├── a1/
│   ├── 9c4e6f8a0b...
│   └── b2d4f6a8c0...
└── ff/
    └── 1234567890...
```

### Exploring a Real Git Repository

You can inspect Git's internals yourself:

```bash
mkdir test-repo && cd test-repo && git init
echo "Hello" > hello.txt
git add hello.txt && git commit -m "First commit"

find .git/objects -type f          # See stored objects
git cat-file -t <hash>            # Show type: blob, tree, or commit
git cat-file -p <hash>            # Show contents
cat .git/HEAD                     # See what HEAD points to
cat .git/refs/heads/main          # See branch pointer
```

---

## What You'll Build

PES-VCS implements five commands across four phases:

```
pes init              Create .pes/ repository structure
pes add <file>...     Stage files (hash + update index)
pes status            Show modified/staged/untracked files
pes commit -m <msg>   Create commit from staged files
pes log               Walk and display commit history
```

The `.pes/` directory structure:

```
my_project/
├── .pes/
│   ├── objects/          # Content-addressable blob/tree/commit storage
│   │   ├── 2f/
│   │   │   └── 8a3b...   # Sharded by first 2 hex chars of hash
│   │   └── a1/
│   │       └── 9c4e...
│   ├── refs/
│   │   └── heads/
│   │       └── main      # Branch pointer (file containing commit hash)
│   ├── index             # Staging area (text file)
│   └── HEAD              # Current branch reference
└── (working directory files)
```

### Architecture Overview

```
┌───────────────────────────────────────────────────────────────┐
│                      WORKING DIRECTORY                        │
│                  (actual files you edit)                       │
└───────────────────────────────────────────────────────────────┘
                              │
                        pes add <file>
                              ▼
┌───────────────────────────────────────────────────────────────┐
│                           INDEX                               │
│                (staged changes, ready to commit)              │
│                100644 a1b2c3... src/main.c                    │
└───────────────────────────────────────────────────────────────┘
                              │
                       pes commit -m "msg"
                              ▼
┌───────────────────────────────────────────────────────────────┐
│                       OBJECT STORE                            │
│  ┌───────┐    ┌───────┐    ┌────────┐                         │
│  │ BLOB  │◄───│ TREE  │◄───│ COMMIT │                         │
│  │(file) │    │(dir)  │    │(snap)  │                         │
│  └───────┘    └───────┘    └────────┘                         │
│  Stored at: .pes/objects/XX/YYY...                            │
└───────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌───────────────────────────────────────────────────────────────┐
│                           REFS                                │
│       .pes/refs/heads/main  →  commit hash                    │
│       .pes/HEAD             →  "ref: refs/heads/main"         │
└───────────────────────────────────────────────────────────────┘
```

---

## Phase 1: Object Storage Foundation

**Filesystem Concepts:** Content-addressable storage, directory sharding, atomic writes, hashing for integrity

**Files:** `pes.h` (read), `object.c` (implement `object_write` and `object_read`)

### What to Implement

Open `object.c`. Two functions are marked `// TODO`:

1. **`object_write`** — Stores data in the object store.
   - Prepends a type header (`"blob <size>\0"`, `"tree <size>\0"`, or `"commit <size>\0"`)
   - Computes SHA-256 of the full object (header + data)
   - Writes atomically using the temp-file-then-rename pattern
   - Shards into subdirectories by first 2 hex chars of hash

2. **`object_read`** — Retrieves and verifies data from the object store.
   - Reads the file, parses the header to extract type and size
   - **Verifies integrity** by recomputing the hash and comparing to the filename
   - Returns the data portion (after the `\0`)

Read the detailed step-by-step comments in `object.c` before starting.

### Testing

```bash
make test_objects
./test_objects
```

The test program verifies:
- Blob storage and retrieval (write, read back, compare)
- Deduplication (same content → same hash → stored once)
- Integrity checking (detects corrupted objects)

**📸 Screenshot 1A:** Output of `./test_objects` showing all tests passing.
<img width="940" height="311" alt="image" src="https://github.com/user-attachments/assets/8d27fb3a-be90-4437-919c-47f8a36ed143" />


**📸 Screenshot 1B:** `find .pes/objects -type f` showing the sharded directory structure.
<img width="940" height="264" alt="image" src="https://github.com/user-attachments/assets/daea532d-3808-4da8-8d85-d71fe428257d" />

---

## Phase 2: Tree Objects

**Filesystem Concepts:** Directory representation, recursive structures, file modes and permissions

**Files:** `tree.h` (read), `tree.c` (implement all TODO functions)

### What to Implement

Open `tree.c`. Implement the function marked `// TODO`:

1. **`tree_from_index`** — Builds a tree hierarchy from the index.
   - Handles nested paths: `"src/main.c"` must create a `src` subtree
   - This is what `pes commit` uses to create the snapshot
   - Writes all tree objects to the object store and returns the root hash

### Testing

```bash
make test_tree
./test_tree
```

The test program verifies:
- Serialize → parse roundtrip preserves entries, modes, and hashes
- Deterministic serialization (same entries in any order → identical output)

**📸 Screenshot 2A:** Output of `./test_tree` showing all tests passing.
<img width="940" height="264" alt="image" src="https://github.com/user-attachments/assets/9d8758cd-60de-4f32-b1cf-0212cf470b63" />

**📸 Screenshot 2B:** Pick a tree object from `find .pes/objects -type f` and run `xxd .pes/objects/XX/YYY... | head -20` to show the raw binary format.
<img width="940" height="81" alt="image" src="https://github.com/user-attachments/assets/04a9e942-6cf6-451d-8b87-33102dba03fd" />

---

## Phase 3: The Index (Staging Area)

**Filesystem Concepts:** File format design, atomic writes, change detection using metadata

**Files:** `index.h` (read), `index.c` (implement all TODO functions)

### What to Implement

Open `index.c`. Three functions are marked `// TODO`:

1. **`index_load`** — Reads the text-based `.pes/index` file into an `Index` struct.
   - If the file doesn't exist, initializes an empty index (this is not an error)
   - Parses each line: `<mode> <hash-hex> <mtime> <size> <path>`

2. **`index_save`** — Writes the index atomically (temp file + rename).
   - Sorts entries by path before writing
   - Uses `fsync()` on the temp file before renaming

3. **`index_add`** — Stages a file: reads it, writes blob to object store, updates index entry.
   - Use the provided `index_find` to check for an existing entry

`index_find` , `index_status` and `index_remove` are already implemented for you — read them to understand the index data structure before starting.

#### Expected Output of `pes status`

```
Staged changes:
  staged:     hello.txt
  staged:     src/main.c

Unstaged changes:
  modified:   README.md
  deleted:    old_file.txt

Untracked files:
  untracked:  notes.txt
```

If a section has no entries, print the header followed by `(nothing to show)`.

### Testing

```bash
make pes
./pes init
echo "hello" > file1.txt
echo "world" > file2.txt
./pes add file1.txt file2.txt
./pes status
cat .pes/index    # Human-readable text format
```

**📸 Screenshot 3A:** Run `./pes init`, `./pes add file1.txt file2.txt`, `./pes status` — show the output.
<img width="940" height="820" alt="image" src="https://github.com/user-attachments/assets/21405e93-ef9f-4f3d-8990-8ce613698387" />

**📸 Screenshot 3B:** `cat .pes/index` showing the text-format index with your entries.
<img width="940" height="173" alt="image" src="https://github.com/user-attachments/assets/54eee5d9-4896-4aac-a4a1-ff500f48b185" />

---

## Phase 4: Commits and History

**Filesystem Concepts:** Linked structures on disk, reference files, atomic pointer updates

**Files:** `commit.h` (read), `commit.c` (implement all TODO functions)

### What to Implement

Open `commit.c`. One function is marked `// TODO`:

1. **`commit_create`** — The main commit function:
   - Builds a tree from the index using `tree_from_index()` (**not** from the working directory — commits snapshot the staged state)
   - Reads current HEAD as the parent (may not exist for first commit)
   - Gets the author string from `pes_author()` (defined in `pes.h`)
   - Writes the commit object, then updates HEAD

`commit_parse`, `commit_serialize`, `commit_walk`, `head_read`, and `head_update` are already implemented — read them to understand the commit format before writing `commit_create`.

The commit text format is specified in the comment at the top of `commit.c`.

### Testing

```bash
./pes init
echo "Hello" > hello.txt
./pes add hello.txt
./pes commit -m "Initial commit"

echo "World" >> hello.txt
./pes add hello.txt
./pes commit -m "Add world"

echo "Goodbye" > bye.txt
./pes add bye.txt
./pes commit -m "Add farewell"

./pes log
```

You can also run the full integration test:

```bash
make test-integration
```

**📸 Screenshot 4A:** Output of `./pes log` showing three commits with hashes, authors, timestamps, and messages.
<img width="940" height="588" alt="image" src="https://github.com/user-attachments/assets/616e88b1-77d0-4737-a7c8-d170d0d2eaf9" />

**📸 Screenshot 4B:** `find .pes -type f | sort` showing object store growth after three commits.
<img width="940" height="299" alt="image" src="https://github.com/user-attachments/assets/1992844e-bf54-41c8-bc8f-0872e2e454a4" />

**📸 Screenshot 4C:** `cat .pes/refs/heads/main` and `cat .pes/HEAD` showing the reference chain.
<img width="940" height="127" alt="image" src="https://github.com/user-attachments/assets/b1323bd8-001e-4618-8b78-bbb1c4e882e2" />

---

## Phase 5 — Branching and Checkout

### Q5.1 — How would you implement `pes checkout <branch>`?

**Files that must change in `.pes/`:**

1. `.pes/HEAD` — rewrite the line from `ref: refs/heads/<current>` to
   `ref: refs/heads/<target>`.
2. If the target branch is new, create `.pes/refs/heads/<target>` pointing to
   the desired commit hash.

**Working-directory update:**

1. Read the target branch tip from `.pes/refs/heads/<branch>`.
2. `object_read` the root tree of that commit.
3. Recursively walk the tree: for blob entries write (or overwrite) the file in
   the working directory; for tree entries (mode `040000`) create directories.
4. Delete any working-directory file that is present in the *current* HEAD's
   tree but absent from the target tree.

**What makes this complex:**

- **Dirty-file conflicts.** If a tracked file has local modifications and the
  target branch has a different version of that file, checkout must refuse or
  the user's edits are destroyed silently.
- **Untracked-file collisions.** If an untracked file sits at a path that the
  target tree would create, checkout must refuse to avoid clobbering it.
- **Partial failure.** A crash halfway through leaves a mixed working directory.
  A production implementation writes a lock or in-progress marker to allow
  recovery.
- **Three-way diff.** Deciding which files to touch requires comparing the
  current-HEAD tree, the target-HEAD tree, and the index all at once.

### Q5.2 — Detecting dirty working-directory conflicts

Using only the index and the object store:

1. Load the current index (the staged snapshot matching HEAD after the last
   commit).
2. For each entry, `stat()` the working-directory file. If `st_mtime` or
   `st_size` differs from the stored values, re-hash the file (SHA-256) and
   compare to the stored hash. A mismatch means the file is dirty.
3. For each path in the target branch's tree: if that path is in the index and
   is dirty (step 2), refuse checkout for that path.
4. For untracked files: if a path in the target tree does not appear in the
   index but already exists on disk, refuse checkout to avoid overwriting it.

The mtime/size check is a fast first pass; the full hash re-read is the
authoritative check used only when metadata differs.

### Q5.3 — Detached HEAD and recovery

When HEAD holds a raw commit hash instead of `ref: refs/heads/<branch>`,
`head_update` writes new hashes directly into HEAD. Commits still chain
correctly via parent pointers, but no branch file is updated. Once you check
out a branch, HEAD is overwritten and the detached chain has no ref pointing
to it — it becomes unreachable from all named references.

**Recovery before switching away:**

```bash
cat .pes/HEAD                            # note the hash
echo <hash> > .pes/refs/heads/recovered
echo "ref: refs/heads/recovered" > .pes/HEAD
```

**Recovery after switching away:** the commit objects still exist in the object
store. Walk every file under `.pes/objects/`, call `object_read` on each, and
look for `OBJ_COMMIT` objects whose parent chain leads to your lost work. Git
calls this `git fsck --lost-found`. The GC grace period (see Q6.2) keeps the
objects safe for at least two weeks.

---

## Phase 6 — Garbage Collection

### Q6.1 — Algorithm to find and delete unreachable objects

**Mark-and-sweep:**

*Mark phase — build the reachable set:*

1. Enumerate every file under `.pes/refs/` and read `HEAD` to get the GC roots.
2. For each root commit hash, call `object_read`. Add its hash, the tree hash,
   and every blob/subtree hash found by recursively walking the tree to a
   `reachable` set. Follow each commit's parent pointer until `has_parent == 0`.

*Data structure:* a sorted array of `ObjectID` (32 bytes each) with binary
search — O(n log n) to build, O(log n) per lookup. A hash table gives O(1)
average lookup for larger repos.

*Sweep phase — delete unreachable objects:*

Walk every file under `.pes/objects/XX/`. Reconstruct the full hash from the
directory name and filename, convert with `hex_to_hash`. If not in `reachable`,
call `unlink()`.

**Estimate for 100,000 commits, 50 branches:**

Assuming roughly four unique objects per commit on average (one commit, one to
two trees, one to two new blobs, the rest shared):

- Reachable set ≈ 400,000 objects → 400,000 `object_read` calls in the mark
  phase.
- Sweep: walk all ~400,000 files in the object store.
- Total: roughly **800,000 file accesses**.

### Q6.2 — Race condition between GC and a concurrent commit

**The race:**

| Time | `pes commit` | GC |
|------|-------------|-----|
| t1 | `object_write(OBJ_BLOB)` stores blob B | — |
| t2 | — | Mark phase reads HEAD → old commit. B is not reachable from any ref. |
| t3 | `object_write(OBJ_TREE)` creates tree T referencing B | — |
| t4 | — | Sweep: B is not in reachable set → `unlink(B)` |
| t5 | `object_write(OBJ_COMMIT)` + `head_update` completes | — |

Result: HEAD now points to a commit whose tree references the deleted blob B.
The repository is corrupt.

**How Git avoids this:**

1. **Grace period (`gc.pruneExpire`, default 2 weeks).** GC skips any loose
   object whose filesystem `mtime` is younger than the grace period. A freshly
   written blob is always newer than two weeks, so it is never collected before
   it is referenced by a commit.
2. **Keep-alive refs.** Git writes temporary refs (`FETCH_HEAD`, `MERGE_HEAD`,
   `ORIG_HEAD`) during in-flight operations, so GC sees those objects as
   reachable roots.
3. **Pack-before-prune ordering.** When converting loose objects to pack files,
   Git writes and verifies the pack before deleting any loose objects, so there
   is never a window where an object exists only in a half-written pack.
4. The atomic rename used when writing objects does not help here — blob B is
   fully written before GC runs. Only the grace period closes the time window
   reliably.

---

