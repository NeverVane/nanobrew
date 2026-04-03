# Error Handling & Performance Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix the 5 most impactful silent error handling bugs and implement the 5 highest-leverage performance optimizations, targeting ~30-40% cold install speedup and eliminating all data-loss-class silent failures.

**Architecture:** Two parallel workstreams (error handling + performance) touching mostly non-overlapping files. The one shared file (`db/database.zig`) is handled in a single task that addresses both the error handling and performance concerns together.

**Tech Stack:** Zig 0.15.2, `zig build test` / `zig build test-security`

---

## Parallelization Strategy

### Stream A: Error Handling (5 tasks)
Touches: `src/db/database.zig`, `src/main.zig` (remove path), `src/cask/install.zig`, `src/main.zig` (install phase marking)

### Stream B: Performance (5 tasks)
Touches: `src/net/downloader.zig`, `src/resolve/deps.zig`, `src/platform/placeholder.zig`, `src/main.zig` (progress renderer), `src/extract/tar.zig`

### Shared task (run first, before either stream)
Task 1: `src/db/database.zig` — both save() error reporting AND batch save optimization

### Safe to parallelize after Task 1
- Stream A tasks 2-5 and Stream B tasks 2-5 touch different files

---

## Task 1: Database save() error reporting + batch save [SHARED — run first]

**Files:**
- Modify: `src/db/database.zig`
- Modify: `src/main.zig` (the post-install recording loop)

**Error handling fix:** `close()` at line 195 does `self.save() catch {}`. If save fails (disk full, permissions), the entire in-memory state is silently lost. Fix: print a warning to stderr on save failure.

**Performance fix:** `recordInstall` calls `save()` after each package. For 10 packages, that's 10 full serializations + 10 fsyncs. Fix: make save() callable explicitly, batch all records then save once.

- [ ] **Step 1: Fix `close()` to warn on save failure**

In `src/db/database.zig`, change line 195 from:
```zig
pub fn close(self: *Database) void {
    self.save() catch {};
}
```
To:
```zig
pub fn close(self: *Database) void {
    self.save() catch |err| {
        std.fs.File.stderr().deprecatedWriter().print("nb: WARNING: failed to save package database: {}\n", .{err}) catch {};
    };
}
```

- [ ] **Step 2: Add `saveIfDirty` pattern — skip save when no mutations happened**

Add a `dirty: bool` field to the Database struct, initialized to `false`. Set it to `true` in `recordInstall`, `recordRemoval`, `recordCaskInstall`, `recordCaskRemoval`, `recordDebInstall`, `recordDebRemoval`, `setPinned`. In `save()`, early-return if `!self.dirty`. Reset `dirty = false` after successful save.

This means the post-install loop can call `recordInstall` N times (each sets dirty=true but doesn't save), then `close()` does one save at the end.

Remove the `self.save() catch {};` calls at the end of each `record*` method — let `close()` handle it.

- [ ] **Step 3: Run tests**

```bash
cd /Users/zanobi/nanobrew && zig build test && zig build
```

- [ ] **Step 4: Commit**

```bash
git commit -am "fix+perf: warn on database save failure, batch saves with dirty flag"
```

---

## Stream A: Error Handling

### Task A2: Fix `nb remove` reporting success when operations fail

**Files:**
- Modify: `src/main.zig` (the remove command handler, around line 740)

Currently:
```zig
nb.linker.unlinkKeg(name, keg.version) catch {};
nb.cellar.remove(name, keg.version) catch {};
db.recordRemoval(name, alloc) catch {};
stdout.print("==> Removed {s}\n", .{name}) catch {};
```

Fix: check each return, print errors, only print success if all steps succeeded.

```zig
var remove_ok = true;
nb.linker.unlinkKeg(name, keg.version) catch |err| {
    stderr.print("nb: error: failed to unlink {s}: {}\n", .{ name, err }) catch {};
    remove_ok = false;
};
nb.cellar.remove(name, keg.version) catch |err| {
    stderr.print("nb: error: failed to remove keg for {s}: {}\n", .{ name, err }) catch {};
    remove_ok = false;
};
db.recordRemoval(name, alloc) catch |err| {
    stderr.print("nb: error: failed to update database for {s}: {}\n", .{ name, err }) catch {};
    remove_ok = false;
};
if (remove_ok) {
    stdout.print("==> Removed {s}\n", .{name}) catch {};
} else {
    stderr.print("nb: {s} partially removed — check errors above\n", .{name}) catch {};
}
```

- [ ] **Step 1: Read the remove handler in main.zig to find exact lines**
- [ ] **Step 2: Apply the fix**
- [ ] **Step 3: Run tests and commit**

```bash
git commit -am "fix: report errors during nb remove instead of false success (#6 audit)"
```

---

### Task A3: Fix install phase not marked as failed on relocate/link errors

**Files:**
- Modify: `src/main.zig` (inside `fullInstallOne`, around line 667-678)

Currently relocate and link failures print to stderr but don't set `had_error` or `phase = .failed`. The progress UI shows a checkmark and the package is recorded as successfully installed.

Fix: on relocate or link failure, mark the phase as failed and set `had_error`.

- [ ] **Step 1: Read `fullInstallOne` to find the exact error handling pattern**
- [ ] **Step 2: After the relocate catch block, add `had_error.store(true, .release)` and return**
- [ ] **Step 3: Same for link failure**
- [ ] **Step 4: Run tests and commit**

```bash
git commit -am "fix: mark install as failed when relocate or link fails (#13 audit)"
```

---

### Task A4: Fix .pkg installer failure not propagated in cask install

**Files:**
- Modify: `src/cask/install.zig` (around line 259)

Currently the `.pkg` handler prints an error but doesn't set `any_artifact_failed = true`. The `.app` handler correctly sets it. One-line fix.

- [ ] **Step 1: Read the .pkg handler to find the exact line**
- [ ] **Step 2: Add `any_artifact_failed = true;` after the error print**
- [ ] **Step 3: Run tests and commit**

```bash
git commit -am "fix: propagate .pkg installer failure in cask install (#12 audit)"
```

---

### Task A5: Fix deb install/remove DB write failures silent

**Files:**
- Modify: `src/main.zig` (deb install path ~line 3108, deb remove path ~line 3161)

Both `recordDebInstall() catch {}` and `recordDebRemoval() catch {}` silently swallow errors. Fix: print a warning.

- [ ] **Step 1: Find both locations**
- [ ] **Step 2: Change `catch {}` to `catch |err| { stderr.print("nb: warning: ...", .{name, err}) catch {}; }`**
- [ ] **Step 3: Run tests and commit**

```bash
git commit -am "fix: warn when deb install/remove database writes fail (#4 #5 audit)"
```

---

## Stream B: Performance

### Task B2: Share TLS client across download threads

**Files:**
- Modify: `src/net/downloader.zig` (the `downloadOne` function and its callers)

Currently each `downloadOne` call creates `var client: std.http.Client = .{ .allocator = alloc }` — a new TLS session per thread. The resolver already shares a client correctly via `fetchFormulaWithClient`.

Fix: create one `std.http.Client` in `downloadAll`, pass it into each worker thread. Since `std.http.Client` in Zig 0.15 is not thread-safe for concurrent requests, the fix is to create one client PER THREAD but reuse it across multiple downloads on the same thread (if a thread handles multiple packages). Alternatively, create a pool of clients equal to the thread count.

Actually, looking at the code: `downloadAll` spawns one thread per package (each calling `downloadOne`). Each thread does exactly one download then exits. So sharing a client between threads doesn't help — each thread needs its own client anyway.

The real fix: reduce to a fixed number of worker threads (e.g., 4-8) that each process multiple packages from a shared queue, reusing their client for each download. This is the `ParallelDownloader` pattern.

- [ ] **Step 1: Read `downloadAll` and understand the threading model**
- [ ] **Step 2: Change from 1-thread-per-package to N-worker-threads with a shared work queue**
- [ ] **Step 3: Each worker creates one client and reuses it for all its packages**
- [ ] **Step 4: Run tests and commit**

```bash
git commit -am "perf: reuse TLS connections across downloads with worker pool"
```

---

### Task B3: Fix O(V²×E) topological sort

**Files:**
- Modify: `src/resolve/deps.zig` (the `topologicalSort` function)

Two fixes:
1. Replace `queue.orderedRemove(0)` (O(N) shift) with a proper queue or `swapRemove`
2. Build a reverse-adjacency map so the inner loop only visits direct dependents, not all edges

- [ ] **Step 1: Read `topologicalSort`**
- [ ] **Step 2: Build `reverse_edges: StringHashMap(ArrayList([]const u8))` during setup**
- [ ] **Step 3: Replace the inner edge scan with `reverse_edges.get(sorted_name)`**
- [ ] **Step 4: Replace `orderedRemove(0)` with index-based queue (front pointer)**
- [ ] **Step 5: Run tests and commit**

```bash
git commit -am "perf: O(V+E) topological sort with reverse adjacency map"
```

---

### Task B4: Single-pass placeholder replacement (eliminate 3 allocs per call)

**Files:**
- Modify: `src/platform/placeholder.zig` (the `replacePlaceholders` function, lines 17-23)

Currently does 3 `replaceOwned` calls (3 allocs, 2 frees). Replace with a single-pass scan using a stack buffer, matching the pattern already used in `relocateTextFile`.

- [ ] **Step 1: Read the current `replacePlaceholders`**
- [ ] **Step 2: Implement single-pass replacement with a `[4096]u8` stack buffer**
- [ ] **Step 3: Run tests and commit**

```bash
git commit -am "perf: single-pass placeholder replacement eliminates 3 allocs per call"
```

---

### Task B5: Buffer progress renderer output (eliminate per-byte syscalls)

**Files:**
- Modify: `src/main.zig` (the `renderProgress` function, around line 519-601)

The padding loop `while (pad > 0) : (pad -= 1) stdout_file.writeAll(" ")` does one syscall per space character. Buffer the entire frame into a stack `[4096]u8` and emit in one `writeAll`.

- [ ] **Step 1: Read `renderProgress`**
- [ ] **Step 2: Replace individual writeAll calls with writes to a stack buffer**
- [ ] **Step 3: Flush the buffer in one writeAll at the end of each frame**
- [ ] **Step 4: Run tests and commit**

```bash
git commit -am "perf: buffer progress renderer output to eliminate per-byte syscalls"
```

---

### Task B6: Wire native tar parser into extraction path [BONUS — highest impact]

**Files:**
- Modify: `src/extract/tar.zig` (replace subprocess tar with native parser)

This is the biggest single performance win but also the most complex. The native parser in `native_tar.zig` is production-ready. The missing piece is a gzip decompression wrapper — `native_tar.extractToDir` expects raw tar data, but bottles are `.tar.gz`.

- [ ] **Step 1: Read `tar.zig:extractTarGz` and `native_tar.zig:extractToDir`**
- [ ] **Step 2: Add gzip decompression using `std.compress.gzip` or `std.compress.flate`**
- [ ] **Step 3: Decompress into memory (or via streaming), then call `native_tar.extractToDir`**
- [ ] **Step 4: Keep the subprocess path as a fallback for non-gzip formats**
- [ ] **Step 5: Run tests extensively — extraction correctness is critical**
- [ ] **Step 6: Commit**

```bash
git commit -am "perf: use native tar parser instead of subprocess for bottle extraction"
```

---

## Execution Order

```
Task 1 (shared DB fix) ────────────┐
                                    ├── Then parallel:
                                    │
                          Stream A  │  Stream B
                          ────────  │  ────────
                          A2 remove │  B2 TLS pool
                          A3 phase  │  B3 topo sort
                          A4 .pkg   │  B4 placeholder
                          A5 deb    │  B5 progress
                                    │  B6 native tar (bonus)
```

Tasks A2-A5 can run as one agent. Tasks B2-B5 can run as another agent. Task B6 (native tar) should be a separate agent due to complexity.
