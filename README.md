# Time-Traveling File System — COL106 Data Structures & Algorithms Assignment

Academic abstract
This repository contains an in-memory, single-process implementation of a versioned file system intended for pedagogical demonstration of data structure design in systems that track file history. A single file is modeled as a tree of versions (linear ancestry in the present implementation) with snapshot semantics; global indices provide quick access to recently modified files and the files with the most versions. The implementation emphasizes custom lightweight containers and heap indices to illustrate trade-offs typical in systems assignments.

Repository contents
- Problem Statement.pdf — the original assignment description and constraints (included).
- index.cpp — the primary implementation and CLI entry-point.
- headers.hpp — declarations for all types and global objects.
- README.md — this document (academic-style overview and usage).

High-level overview
- Each managed file is represented by a File object that maintains:
  - A root TreeNode that represents the initial version (version_id 0).
  - A version map (intMap) mapping version IDs to nodes.
  - An active_version pointer denoting the current visible version.
- Versions (TreeNode) contain:
  - content, created timestamp, optional snapshot timestamp and message, parent pointer, children vector.
- Two binary min-heaps provide global indices:
  - RECENT_FILES — keyed by last modified timestamp (min-heap to fetch least-recent? See details below).
  - BIGGEST_TREES — keyed by total_versions (min-heap to fetch smallest total_versions first, allowing read(k) to obtain k "smallest" items).
- Custom maps:
  - strMap (string → File*) and intMap (int → TreeNode*) using separate chaining hashing to illustrate a simple hash-table implementation.

Design and data structures (detailed)
- TreeNode (struct)
  - version_id: int
  - content: string
  - message: string (snapshot message)
  - created_timestamp, snapshot_timestamp: time_t
  - parent: TreeNode*
  - children: vector<TreeNode*>
  - Rationale: TreeNode models a version graph; the implementation currently organizes versions in a singly-linked ancestry (parent pointers), with possible branching via children (vector) if multiple versions are created from one node.

- File (class)
  - root: TreeNode* (version 0)
  - active_version: TreeNode*
  - version_map: intMap (quick lookup by version ID)
  - total_versions: int
  - modified_time: time_t (updated on edit)
  - indexInHeap: int (location in heap arrays; used by binaryMinHeap)
  - Major operations:
    - read(): returns content of active_version.
    - insert(content): appends content; if the current active version is snapshotted then a new child version is created instead (to preserve immutability of snapshots).
    - update(content): replaces content of active_version, or creates a new child version if active is snapshotted.
    - snapShot(message): marks the active_version as a snapshot with a timestamp and optional message.
    - rollBack(versionId): set active_version to specified version (or parent if id == -1).
    - history(): prints version metadata up to the root.

- intMap / strMap
  - Simple hash maps with separate chaining. The hash functions are intentionally minimal (sum of bytes for strMap and modulo for intMap) for pedagogical clarity.

- binaryMinHeap
  - Array-backed binary min-heap storing file names, where keys are either:
    - modified time (RECENT_FILES) or
    - total_versions (BIGGEST_TREES).
  - Implementation provides addNode, HeapifyGetMin (pop-min), modifyKey (decrease/increase key with subsequent heapify), and read(k) which returns the top k items (via temporary removal and reinsertion).
  - Note: The constructor parameter isDataTotalCount is inverted internally so as to select the stored key logic.

Command-line interface (CLI)
- The program is an interactive console application. On startup it prints the valid commands and then accepts free-form commands on stdin.
- Supported commands:
  - CREATE <filename>
  - READ <filename>
  - INSERT <filename> <content>         (appends; if active version is a snapshot, a new version is created)
  - UPDATE <filename> <content>         (replaces content; if active version is a snapshot, a new version is created)
  - SNAPSHOT <filename> <message>       (marks the active version as snapshotted with message)
  - ROLLBACK <filename> [versionID]     (if versionID omitted, rolls back to parent of active)
  - HISTORY <filename>                  (prints version_id, created, snapshot timestamp and snapshot message along the ancestry)
  - RECENT_FILES <ignored> <num>        (prints num entries from RECENT_FILES heap)
  - BIGGEST_TREES <ignored> <num>       (prints num entries from BIGGEST_TREES heap)
- Parsing notes:
  - The program tokenizes the first two tokens (command and filename). For content or messages, it uses the substring of the original input beginning at the first token of the content/message; therefore multi-word content messages are supported.
  - On INSERT/UPDATE, if the active version is already snapshotted, the code either prompts (if the snapshot has no message) or creates a child version and stores content there.

Usage examples
- Build (see the Build section for compilation details)
- Example session (user input lines shown; prompts printed by the program):
  - CREATE fileA
  - INSERT fileA This is the first line of fileA
  - INSERT fileA Another line appended to fileA
  - SNAPSHOT fileA Finalizing version 1 before branching
  - INSERT fileA Additional content in a new version
  - HISTORY fileA
    - Output: a table of version IDs, created timestamps, snapshot timestamps and messages (printed from current active to root)
  - ROLLBACK fileA 1
    - Sets active version to version id 1
  - READ fileA
    - Prints content of active version

Build and run
- Compiler: GNU g++ or clang++ supporting C++11 or later.
- The program is interactive and listens for commands until interrupted (Ctrl+C).

Algorithmic complexity (informal)
- Hash map operations (strMap / intMap): expected O(1) average for insert, lookup (depends on hash quality and load).
- Version creation (addChild): O(1) to allocate and link a TreeNode; storing it in version_map is expected O(1).
- Heap operations:
  - addNode, modifyKey, HeapifyGetMin: O(log n) where n is number of files in the heap.
  - read(k): *O(k log n)* (the implementation pops k items and reinserts them).
- History traversal: *O(h)* to traverse h ancestors.
- Memory:
  - All data structures are kept in memory. Each File maintains all versions persistently for the process lifetime.

Correctness and behavioral notes
- Snapshot semantics: when a version has a non-zero snapshot_timestamp it is treated as immutable; subsequent INSERT or UPDATE will create a new version rather than mutate the snapshot.
- Version IDs: assigned incrementally per File starting at 0 for the root.
- History prints entries from active_version back to root (active first, then parent, and so on).
- RECENT_FILES and BIGGEST_TREES are min-heaps; the semantics of "read k" print the k smallest elements according to the heap key. Adjust interpretation accordingly when using those commands.

Potential extensions (for further study)
- Add persistence (serialize the file/version trees and heaps to disk).
- Replace simple hash functions with a robust std::unordered_map or an improved custom hash.
- Implement a max-heap variant (or configurable comparator) for RECENT_FILES / BIGGEST_TREES to match expected "top-k" semantics (most recent / largest).
- Add unit tests and automated test harness (current repository has no test suite).
- Support branching explicitly and merging of branches with conflict resolution strategies.
- Pretty-print timestamps (e.g., ISO 8601) and provide more metadata in HISTORY output.

Academic usage / grading guidance
- The code is designed to demonstrate concrete data-structure choices. Evaluate correctness by inspecting:
  - That version creation and snapshot behaviors follow the described semantics.
  - That global indices respond correctly to modifications (modifyKey is called after writes).
  - That history traversal returns the expected ancestry ordering.
- For grading, consider unit tests that programmatically:
  - Create files, perform updates and snapshots, then assert the content and timestamps.
  - Validate that RECENT_FILES and BIGGEST_TREES read operations reflect state changes.

References
- Problem Statement.pdf (included).
