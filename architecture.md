# nanobrew — Architecture Overview

> Generated from source inspection of [`NeverVane/nanobrew`](https://github.com/NeverVane/nanobrew)  
> Current version: **0.1.083** | Language: **Zig 0.15+** | Binary size: **~1.2 MB**

---

## Table of Contents

1. [Project Summary](#1-project-summary)
2. [Repository Layout](#2-repository-layout)
3. [Module-by-Module Primer](#3-module-by-module-primer)
   - [Entry Point — `src/main.zig`](#31-entry-point--srcmainzig)
   - [Library Root — `src/root.zig`](#32-library-root--srcrootzig)
   - [API Layer — `src/api/`](#33-api-layer--srcapi)
   - [Resolver — `src/resolve/`](#34-resolver--srcresolve)
   - [Network — `src/net/`](#35-network--srcnet)
   - [Store & Cache — `src/store/`](#36-store--cache--srcstore)
   - [Extraction — `src/extract/`](#37-extraction--srcextract)
   - [Cellar — `src/cellar/`](#38-cellar--srccellar)
   - [Linker — `src/linker/`](#39-linker--srclinker)
   - [Platform Abstraction — `src/platform/`](#310-platform-abstraction--srcplatform)
   - [Binary Relocation — `src/macho/` & `src/elf/`](#311-binary-relocation--srcmacho--srcelf)
   - [Database — `src/db/`](#312-database--srcdb)
   - [Cask Pipeline — `src/cask/`](#313-cask-pipeline--srccask)
   - [Deb Pipeline — `src/deb/`](#314-deb-pipeline--srcdeb)
   - [Services — `src/services/`](#315-services--srcservices)
   - [Build Pipeline — `src/build/`](#316-build-pipeline--srcbuild)
   - [Kernel Primitives — `src/kernel/`](#317-kernel-primitives--srckernel)
   - [Memory — `src/mem/`](#318-memory--srcmem)
   - [Execution Primitives — `src/exec/`](#319-execution-primitives--srcexec)
   - [Version — `src/version.zig`](#320-version--srcversionzig)
   - [Security Tests — `src/security_test.zig`](#321-security-tests--srcsecurity_testzig)
   - [Cloudflare Worker — `worker/`](#322-cloudflare-worker--worker)
4. [Dependency Graph](#4-dependency-graph)
5. [Install Pipeline Walkthrough](#5-install-pipeline-walkthrough)
6. [Runtime Directory Layout](#6-runtime-directory-layout)
7. [Build System](#7-build-system)
8. [CI / CD](#8-ci--cd)
9. [Adoption Checklist](#9-adoption-checklist)

---

## 1. Project Summary

nanobrew is a **drop-in Homebrew client** written in Zig. It re-uses Homebrew's formula, bottle, and cask ecosystem but replaces the Ruby-based CLI with a single 1.2 MB static binary. Key differentiators:

| Property | Value |
|----------|-------|
| Language | Zig 0.15+ |
| Targets | macOS arm64/x86_64, Linux aarch64/x86_64 (musl static) |
| Binary size | ~1.2 MB |
| Install root | `/opt/nanobrew/` |
| State file | `/opt/nanobrew/db/state.json` |
| Package sources | Homebrew bottles (macOS/Linux), Homebrew casks (macOS), APT `.deb` (Linux) |
| Minimum runtime deps | `tar` (for bottle extraction); `patchelf` on Linux; `codesign` / `hdiutil` on macOS |
| Status | Experimental |

Performance highlights:
- **Warm install** (bottle already cached): ~3.5 ms (Homebrew: 4–14 s)
- **Linux deb warm install**: up to **13× faster** than `apt-get`
- **NBIX binary index cache**: 70K packages deserialized in 32 ms

---

## 2. Repository Layout

```
nanobrew/
├── build.zig               # Zig build script — exe + test targets + cross-compile steps
├── src/
│   ├── main.zig            # CLI entry point (all commands dispatched here)
│   ├── root.zig            # Library module re-exporter
│   ├── version.zig         # Semantic version comparison
│   ├── security_test.zig   # Adversarial security tests
│   ├── api/
│   │   ├── client.zig      # Homebrew JSON API client + response cache
│   │   ├── formula.zig     # Formula struct definition
│   │   ├── cask.zig        # Cask struct definition
│   │   ├── search.zig      # Formula/cask search against Homebrew API
│   │   └── tap.zig         # Third-party tap fetcher + Ruby formula parser
│   ├── resolve/
│   │   └── deps.zig        # BFS dependency resolver, topological sort
│   ├── net/
│   │   ├── fetch.zig       # Low-level HTTP GET (redirect, gzip decompress)
│   │   └── downloader.zig  # Parallel bottle downloader + GHCR token cache
│   ├── store/
│   │   ├── store.zig       # Content-addressable extracted-bottle store
│   │   └── blob_cache.zig  # Downloaded blob path helpers
│   ├── extract/
│   │   ├── tar.zig         # Bottle extraction via system `tar`
│   │   └── native_tar.zig  # Pure-Zig USTAR/GNU tar parser (used for .deb)
│   ├── cellar/
│   │   └── cellar.zig      # Materialize keg from store → Cellar via COW copy
│   ├── linker/
│   │   └── linker.zig      # Create/remove symlinks in prefix/bin/ and opt/
│   ├── platform/
│   │   ├── platform.zig    # Platform detection hub (is_linux, is_macos, deb_arch)
│   │   ├── paths.zig       # Centralized path constants
│   │   ├── copy.zig        # COW copy abstraction (clonefile / cp --reflink)
│   │   ├── relocate.zig    # Dispatch to Mach-O or ELF relocator
│   │   └── placeholder.zig # @@HOMEBREW_*@@ placeholder detection/replacement
│   ├── macho/
│   │   └── relocate.zig    # Mach-O header parser + install_name_tool + codesign
│   ├── elf/
│   │   └── relocate.zig    # ELF header parser + patchelf RPATH fixup
│   ├── db/
│   │   └── database.zig    # JSON state file (kegs, casks, debs, history)
│   ├── cask/
│   │   └── install.zig     # macOS cask install/remove pipeline (.dmg/.zip/.pkg)
│   ├── deb/
│   │   ├── distro.zig      # /etc/os-release parser → APT mirror selection
│   │   ├── index.zig       # APT Packages index parser (RFC 822 format)
│   │   ├── resolver.zig    # Deb dependency resolver + virtual-package handling
│   │   └── extract.zig     # .deb (ar archive) extractor → native_tar
│   ├── services/
│   │   ├── services.zig    # Service dispatcher (launchd vs systemd)
│   │   ├── launchd.zig     # macOS launchctl service discovery + control
│   │   └── systemd.zig     # Linux systemd service discovery + control
│   ├── build/
│   │   ├── source.zig      # Source build pipeline (cmake/autotools/meson/make)
│   │   └── postinstall.zig # post_install script runner + caveat display
│   ├── kernel/
│   │   ├── simd_scanner.zig # Comptime SIMD byte scanner (from zigrep)
│   │   └── mmap_reader.zig  # mmap-backed zero-copy file reader (from zigrep)
│   ├── mem/
│   │   └── arena.zig        # Fixed-size bump allocator (ScratchArena)
│   └── exec/
│       ├── thread_pool.zig  # Chase-Lev work-stealing thread pool
│       └── dir_queue.zig    # Lock-free MPMC work queue (4096-slot ring buffer)
├── worker/
│   └── src/index.js        # Cloudflare Worker serving install.sh at nanobrew.trilok.ai
├── tests/
│   ├── smoke-test.sh       # Integration smoke tests (install/remove/list/info)
│   └── deb-parity.sh       # Linux .deb parity test vs apt-get
├── bench/
│   ├── bench.sh            # Benchmark script
│   └── Dockerfile          # Docker env for Linux deb benchmarks
├── Formula/
│   └── nanobrew.rb         # Homebrew formula for installing nanobrew via `brew`
├── .github/workflows/
│   ├── ci.yml              # CI: build + unit tests + smoke tests on macOS + Linux
│   ├── benchmark.yml       # Weekly auto-benchmark against Homebrew
│   └── release.yml         # Release: cross-compile → tag → GitHub Release + SHA256
├── install.sh              # Local build-from-source installer
├── README.md
├── CHANGELOG.md
├── BENCHMARKS.md
└── SECURITY.md
```

---

## 3. Module-by-Module Primer

### 3.1 Entry Point — `src/main.zig`

**Role:** CLI dispatcher. Parses `argv[1]` into a `Command` enum, calls the corresponding `run*()` function. Also triggers a non-blocking background update check once per day.

**Key types:**
- `Command` enum — 22 commands (`init`, `install`, `remove`, `list`, `info`, `search`, `upgrade`, `update`, `doctor`, `cleanup`, `outdated`, `pin`, `unpin`, `rollback`, `bundle`, `deps`, `services`, `completions`, `nuke`, `migrate`, `reinstall`, `leaves`)
- `Phase` enum — per-package install phases rendered in the live progress UI (`waiting` → `downloading` → `extracting` → `installing` → `relocating` → `linking` → `done` / `failed`)

**Key functions:**
- `runInstall` — validates names, resolves deps, runs up to 16 concurrent `fullInstallOne` threads, renders live spinner UI on TTY
- `fullInstallOne` — per-package: download → extract → materialize → relocate → link → post-install
- `renderProgress` — ANSI escape-code spinner that redraws N lines in-place, hidden on non-TTY
- `isPackageNameSafe` — allowlist validation (alphanumeric + `-_@.+`, ≤256 chars, max 2 slashes for tap refs, no `..`)
- `runDebInstall` / `runCaskInstall` — branch early to specialized pipelines
- `runMigrate` — scans `/opt/homebrew/Cellar` (macOS) or `/home/linuxbrew/.linuxbrew/` (Linux) to import existing packages

**Concurrency model:** Thread-per-package with a sliding window cap of 16 simultaneous threads. Uses `std.atomic.Value(u8)` for phase tracking and `std.atomic.Value(bool)` for error signalling.

---

### 3.2 Library Root — `src/root.zig`

**Role:** Re-exports all modules as a named `nanobrew` library. Drives test discovery with `comptime { _ = module; }` references. Includes architecture comments describing the six design pillars: comptime SIMD, mmap, arena allocators, lock-free MPMC queues, COW copy, and direct kqueue/epoll.

All modules are imported via `@import("nanobrew")` in `src/main.zig`.

---

### 3.3 API Layer — `src/api/`

#### `formula.zig` — Formula struct

```zig
pub const Formula = struct {
    name, version, revision, rebuild: ...,
    desc, dependencies, bottle_url, bottle_sha256,
    source_url, source_sha256, build_deps, caveats,
    post_install_defined: bool,
};
```

- `BOTTLE_TAG`: comptime-selected platform tag (`arm64_sonoma`, `sonoma`, `x86_64_linux`, `aarch64_linux`)
- `BOTTLE_FALLBACKS`: ordered list of alternate tags tried if the primary tag has no bottle
- `effectiveVersion()`: appends `_<rebuild>` suffix when rebuild > 0
- `cellarPath()`: formats `prefix/Cellar/<name>/<version>`

#### `cask.zig` — Cask struct

Represents a macOS app bundle. Fields: `token`, `name`, `version`, `url`, `sha256`, `homepage`, `desc`, `auto_updates`, `artifacts` (`[]Artifact`), `min_macos`.

`Artifact` is a tagged union: `.app`, `.binary{source,target}`, `.pkg`, `.uninstall{quit,pkgutil}`.

`downloadFormat()` infers `.dmg`/`.zip`/`.pkg`/`.tar_gz` from URL suffix.

#### `client.zig` — Homebrew JSON API client

- Fetches `https://formulae.brew.sh/api/formula/<name>.json` with 5-minute file cache at `/opt/nanobrew/cache/api/<name>.json`
- Respects `NANOBREW_API_DOMAIN` / `HOMEBREW_API_DOMAIN` env vars for mirror overrides
- Detects tap refs (2 slashes in name) and routes to `tap.fetchTapFormula`
- `fetchFormulaWithClient()`: allows sharing an `std.http.Client` across calls for TLS reuse
- `parseFormulaJson()`: hand-written JSON parser extracting version, deps, bottle blocks for current `BOTTLE_TAG` + fallbacks
- Cask variant: `fetchCask()` / `parseCaskJson()` fetching from `/api/cask/<token>.json`

#### `search.zig` — Search API

- Downloads `https://formulae.brew.sh/api/formula.json` and `/api/cask.json` (1-hour cache)
- Case-insensitive substring match on name + description
- Returns `[]SearchResult` with name, version, desc, is_cask

#### `tap.zig` — Third-party tap support

- Parses `"user/tap/formula"` refs (exactly 2 slashes)
- Constructs GitHub raw URLs: `raw.githubusercontent.com/<user>/homebrew-<tap>/HEAD/Formula/<name>.rb` with fallbacks for shard layouts and repo-root layout
- `parseRubyFormula()`: line-by-line Ruby DSL parser extracting version, url, sha256, deps, bottle blocks, platform conditionals (`on_macos`/`on_linux`), `#{version}` interpolation
- `parseRubyCask()`: same for cask `.rb` files
- Handles `:recommended`/`:optional` dependencies (skipped), `uses_from_macos` (resolved on macOS), pre-built binary detection

---

### 3.4 Resolver — `src/resolve/`

#### `deps.zig` — BFS dependency resolver

```zig
pub const DepResolver = struct {
    alloc, formulae: StringHashMap(Formula), edges: StringHashMap([][]const u8), client: ?http.Client,
    pub fn resolve(name) !void { ... }   // BFS
    pub fn topologicalSort() ![]Formula { ... }  // Kahn's algorithm
    pub fn hasFormula(name) bool { ... }
};
```

**Algorithm:**
1. Seed frontier with requested package name(s)
2. For each BFS level, fetch all frontier formulas **in parallel** (one thread per unknown dep)
3. Collect results, add new unseen deps to next frontier
4. Repeat until frontier is empty
5. `topologicalSort()` uses Kahn's algorithm (in-degree map + queue) to produce install order; detects cycles (returns `error.CycleDetected`)

**Performance:** Single `http.Client` is shared across serial fetches; parallel fetches each get their own client (stdlib HTTP is not thread-safe) but benefit from OS-level DNS caching.

**Tap short-name handling:** `tapShortName()` strips `user/tap/` prefix for deduplication in the formulae map.

---

### 3.5 Network — `src/net/`

#### `fetch.zig` — Low-level HTTP GET

- Wraps `std.http.Client` with redirect following (up to 5 hops) and auto gzip decompression
- `get(url)` — creates a fresh client per call (use when thread-safety is needed)
- `getWithClient(client, url)` — reuses an existing client for TLS connection reuse
- `download(url, path)` — streaming download to file (used by source builder)

#### `downloader.zig` — Parallel bottle downloader

- `downloadOne()`: downloads a single bottle tarball from GHCR:
  - Obtains a GHCR Bearer token (cached for 4 minutes in `/opt/nanobrew/cache/tokens/`)
  - Applies `NANOBREW_BOTTLE_DOMAIN` / `HOMEBREW_BOTTLE_DOMAIN` URL rewriting if set
  - Streams response through a `Sha256HashedReader` for single-pass SHA256 verification
  - Writes atomically: temp file in `cache/tmp/` → rename to `cache/blobs/<sha256>`
  - Skips if blob already exists

- `ParallelDownloader`: queue-based batch downloader (up to 8 threads via work-stealing)

---

### 3.6 Store & Cache — `src/store/`

#### `store.zig` — Content-addressable extracted store

- `ensureEntry(blob_path, sha256)`: extracts a bottle tarball into `/opt/nanobrew/store/<sha256>/` if not already there
- `hasEntry(sha256)`, `entryPath(sha256)`, `removeEntry(sha256)` — simple filesystem queries
- Delegates to `extract/tar.zig` for actual extraction

#### `blob_cache.zig` — Downloaded blob helpers

- `blobPath(sha256)` → `/opt/nanobrew/cache/blobs/<sha256>` (uses `threadlocal` buffer)
- `has(sha256)` / `evict(sha256)` — cache presence check and invalidation

---

### 3.7 Extraction — `src/extract/`

#### `tar.zig` — System tar wrapper

- `extractToStore(blob_path, sha256)`: creates `store/<sha256>/`, shells out to `tar xzf blob_path -C dest_dir`
- Atomic: errdefers `deleteTree` on the created directory if extraction fails
- Labeled "v0" — v1 plan is mmap + `std.compress.flate` for zero-copy extraction

#### `native_tar.zig` — Pure-Zig USTAR/GNU tar parser

Used for `.deb` extraction (where subprocess `tar` is not reliable). Supports:
- USTAR and GNU tar formats
- Regular files (`'0'`/`'\0'`), directories (`'5'`), symlinks (`'2'`), hardlinks (`'1'`)
- GNU long-name extensions (`'L'`, `'K'`) for paths > 100 chars
- PAX headers (`'g'`, `'x'`) — skipped safely
- **Path traversal protection**: rejects `..` components and absolute paths
- Returns list of extracted file paths (caller owns memory)

`extractToDir(data, dest_dir)` — takes already-decompressed tar bytes in memory.

---

### 3.8 Cellar — `src/cellar/`

#### `cellar.zig` — Keg materialization

- `materialize(sha256, name, version)`:
  1. Locates keg source in `store/<sha256>/<name>/<version>/` (calls `detectStoreVersion` to handle version drift)
  2. Ensures `Cellar/<name>/` parent exists
  3. Removes any existing keg at the destination
  4. Tries `copy.cloneTree()` (macOS APFS `clonefile(2)`, zero-cost COW)
  5. Falls back to `copy.cpFallback()` (`cp --reflink=auto -R` on Linux, `cp -R` on macOS)

- `remove(name, version)`: deletes `Cellar/<name>/<version>/`, removes parent dir if empty

- `detectKegVersion(name, version, buf)`: finds the actual installed version directory (handles version suffix mismatches between API and on-disk)

---

### 3.9 Linker — `src/linker/`

#### `linker.zig` — Symlink management

- `linkKeg(name, version)`:
  - Creates symlinks `Cellar/<name>/<ver>/bin/*` → `prefix/bin/`
  - Creates symlinks `Cellar/<name>/<ver>/sbin/*` → `prefix/bin/`
  - Creates `prefix/opt/<name>` symlink pointing to the keg root

- `unlinkKeg(name, version)`: removes all symlinks pointing into the keg

- Silently overwrites existing symlinks (last writer wins — no conflict detection in current implementation)

---

### 3.10 Platform Abstraction — `src/platform/`

#### `platform.zig`

Exports `is_linux`, `is_macos` booleans and `deb_arch` string (`"arm64"` / `"amd64"`). Re-exports `paths`, `copy`, `relocate`, `placeholder`.

#### `paths.zig`

All path constants as comptime strings:

| Constant | Value |
|----------|-------|
| `ROOT` | `/opt/nanobrew` |
| `PREFIX` | `/opt/nanobrew/prefix` |
| `CELLAR_DIR` | `/opt/nanobrew/prefix/Cellar` |
| `CASKROOM_DIR` | `/opt/nanobrew/prefix/Caskroom` |
| `BIN_DIR` | `/opt/nanobrew/prefix/bin` |
| `OPT_DIR` | `/opt/nanobrew/prefix/opt` |
| `STORE_DIR` | `/opt/nanobrew/store` |
| `BLOBS_DIR` | `/opt/nanobrew/cache/blobs` |
| `TMP_DIR` | `/opt/nanobrew/cache/tmp` |
| `API_CACHE_DIR` | `/opt/nanobrew/cache/api` |
| `APT_CACHE_DIR` | `/opt/nanobrew/cache/apt` |
| `TOKEN_CACHE_DIR` | `/opt/nanobrew/cache/tokens` |
| `DB_PATH` | `/opt/nanobrew/db/state.json` |

Also defines Homebrew placeholder strings (`@@HOMEBREW_PREFIX@@`, etc.) and their real replacements.

#### `copy.zig`

- `cloneTree(src, dst)` — calls macOS `clonefile(2)` syscall; returns `false` on Linux
- `cpFallback(src, dst)` — `cp --reflink=auto -R` on Linux, `cp -R` on macOS

#### `relocate.zig`

Comptime dispatch:
- macOS → `macho/relocate.relocateKeg`
- Linux → `elf/relocate.relocateKeg`

Also exports `replaceKegPlaceholders` from `placeholder.zig`.

#### `placeholder.zig`

- `hasPlaceholder(s)` — scans for `"@@HOMEBREW"`
- `replacePlaceholders(input)` — three-pass string replacement (Cellar → prefix → repository)
- `fileContainsPlaceholder(path)` — streaming scan with overlap buffer (handles needle spanning read boundaries)
- `relocateTextFile(path)` — replaces placeholders in text config files (`.pc`, `.cmake`, `.la`, etc.); skips binary files by magic byte check

---

### 3.11 Binary Relocation — `src/macho/` & `src/elf/`

#### `macho/relocate.zig` — Mach-O relocator (macOS)

Homebrew bottles embed `@@HOMEBREW_PREFIX@@` and `@@HOMEBREW_CELLAR@@` in Mach-O load commands. This module:

1. **Parses Mach-O headers natively** (FAT + 64-bit Mach-O, both endiannesses) — no `otool` subprocess
2. Walks load commands to find `LC_ID_DYLIB`, `LC_LOAD_DYLIB`, `LC_LOAD_WEAK_DYLIB`, `LC_REEXPORT_DYLIB`, `LC_RPATH` — extracts name/path strings
3. If any string contains a placeholder, calls `install_name_tool` to fix it
4. Collects all modified files, calls `codesign --force --sign -` in **one batch call**

Directories scanned: `bin`, `sbin`, `lib`, `libexec`, `Frameworks`

**Process spawn reduction vs old approach:**  
Old: `otool + install_name_tool + codesign` = 3N spawns  
New: `install_name_tool + 1 codesign` = N+1 spawns (3× fewer)

#### `elf/relocate.zig` — ELF relocator (Linux)

Mirrors the Mach-O approach:
1. Detects ELF files by magic bytes (`0x7f ELF`)
2. Parses ELF headers (64-bit, checks `.dynstr` section for placeholders)
3. Uses `patchelf --set-rpath` when RPATH needs updating
4. Replaces placeholders in `.pc`, `.cmake`, `.la`, `.sh`, `.cfg` text files
5. Checks `patchelf` availability upfront — emits actionable error if missing
6. No codesign step needed on Linux

---

### 3.12 Database — `src/db/`

#### `database.zig` — JSON state database

Persistent state at `/opt/nanobrew/db/state.json`. Loaded fully into memory on open, written back atomically on close.

**Structs:**
```zig
Keg        { name, version, sha256, pinned: bool, installed_at: i64 }
CaskRecord { token, version, apps: [][]u8, binaries: [][]u8 }
DebRecord  { name, version, files: [][]u8, sha256, installed_at: i64 }
HistoryEntry { version, sha256, installed_at: i64 }
```

**Key methods:**
- `open(alloc)` — reads and JSON-parses state file; returns empty DB on parse failure
- `close()` — serializes to JSON and writes back
- `recordInstall(name, version, sha256)` — upsert keg + append to history
- `recordRemoval(name)` — remove keg from list
- `findKeg(name)` / `listInstalled()` / `listInstalledCasks()` / `listInstalledDebs()`
- `writeJsonEscaped(writer, s)` — JSON-safe string writer (escapes `\0`, `"`, `\`, control chars)

---

### 3.13 Cask Pipeline — `src/cask/`

#### `cask/install.zig` — macOS app bundle installer

macOS-only (returns `error.CaskNotSupported` on Linux).

Pipeline:
1. Download artifact to `cache/tmp/<token>.<ext>`
2. Create `Caskroom/<token>/<version>/` directory
3. Mount/extract based on format:
   - `.dmg` — removes quarantine xattr, mounts via `hdiutil attach`, copies `.app` bundles
   - `.zip` — extracts via `unzip` subprocess
   - `.tar.gz` — extracts via `tar`
   - `.pkg` — runs `installer -pkg` subprocess
4. Copy `.app` bundles to `/Applications/` (validates app exists — errors instead of silently succeeding)
5. Create binary symlinks for `binary` artifacts in `prefix/bin/`
6. Record in database

Removal: unlinks binaries, deletes from `/Applications/`, removes Caskroom entry, unmounts DMG if needed, records in DB.

---

### 3.14 Deb Pipeline — `src/deb/`

Linux-specific. Implements a full apt-get replacement in pure Zig.

#### `distro.zig` — Distro detection

- Parses `/etc/os-release` for `ID=` and `VERSION_CODENAME=`
- Selects APT mirror: Ubuntu (`archive.ubuntu.com` / `ports.ubuntu.com`) or Debian (`deb.debian.org`)
- Architecture-aware: `arm64` uses `ports.ubuntu.com`

#### `index.zig` — APT Packages index parser

- Parses RFC 822-style `Packages` format (double-newline paragraph blocks)
- Uses an `ArenaAllocator` — single `deinit()` frees all 70K parsed packages
- `ParsedIndex { packages: []DebPackage, arena }` — 32 ms deserialization for Ubuntu full index
- Per-package fields: `name`, `version`, `depends`, `provides`, `filename`, `sha256`, `size`, `description`

#### `resolver.zig` — Deb dependency resolver

- `parseDependsField()`: parses `pkg (>= ver), pkg2 | pkg3` format
  - Handles alternatives (`|`) — picks first available in index or provides_map
  - Falls back to first alternative if none found
- `resolveTransitiveDeps()`: topological sort with virtual-package resolution via `provides_map`
- Virtual packages resolved via `Provides:` field — enables `build-essential` and similar metapackages

#### `extract.zig` — .deb extractor

.deb files are `ar(1)` archives. This module:
1. Parses the `ar` format header (60-byte fixed headers, `!<arch>\n` magic)
2. Locates `data.tar.*` member (supports `gzip`, `zstd`, `xz` compression; `xz` falls back to subprocess)
3. Decompresses in memory using `std.compress`
4. Passes decompressed bytes to `native_tar.extractToDir()`
5. `extractDebToPrefixWithFiles()` — extracts to `/` and returns file list for tracking in DB
6. `runPostinst()` — extracts `control.tar.*`, runs `postinst` script non-fatally

---

### 3.15 Services — `src/services/`

#### `services.zig` — Dispatcher

Comptime dispatch: Linux → `systemd.zig`, macOS → `launchd.zig`. Exports `Service`, `discoverServices`, `isRunning`, `start`, `stop`.

#### `launchd.zig` (macOS)

- Discovers `homebrew.mxcl.*.plist` files within Cellar keg directories
- `isRunning(label)` — calls `launchctl list <label>`
- `start(label)` — `launchctl load <plist_path>`
- `stop(label)` — `launchctl unload <plist_path>`

#### `systemd.zig` (Linux)

Same interface, uses `systemctl` subprocess calls. Scans Cellar for `.service` files.

---

### 3.16 Build Pipeline — `src/build/`

#### `source.zig` — Source build pipeline

Used when no pre-built bottle is available (sets `bottle_url = ""`):
1. Downloads source tarball to `cache/tmp/<name>-<version>.tar.gz`
2. Verifies SHA256 via `shasum -a 256`
3. Detects build system (`cmake`, `autotools`/`./configure`, `meson`, `make`) by directory inspection
4. Runs configure + build + install subprocesses with prefix set to keg path

#### `postinstall.zig` — Post-install hooks

Two concerns handled:
1. **Caveats** — prints the `caveats` field from Formula after install
2. **post_install scripts** — fetches the Ruby formula from homebrew-core, extracts the `def post_install ... end` block, interprets a subset of Ruby commands (mkdir_p, symlink, bin.install, etc.)

---

### 3.17 Kernel Primitives — `src/kernel/`

Imported from the **zigrep** project (same author). Not specific to package management — reusable scanning infrastructure.

#### `simd_scanner.zig`

- `bestSimdWidth(target)` — comptime detection: AVX-512 (64), AVX2 (32), SSE2/NEON (16)
- `ByteScanner(N)` — comptime-parameterized scanner using `@Vector(N, u8)`:
  - `findFirst(haystack, needle)` — SIMD memchr
  - `scanAll(haystack, needle, callback)` — first+last byte filter (Mula technique) before full match
  - Software prefetch (`@prefetch`) to hide memory latency

#### `mmap_reader.zig`

- `MappedFile` — `mmap(PROT_READ, MAP_PRIVATE)` with `MADV_SEQUENTIAL` hint
- `open(path)` → `?MappedFile` (null for empty files or non-mappable fds)
- `close()` → `munmap()`
- `prefetchRange()` — explicit `madvise` for specific byte ranges

Both modules are available to the rest of nanobrew but are primarily used in the deb index parsing path.

---

### 3.18 Memory — `src/mem/`

#### `arena.zig` — ScratchArena

Fixed-size bump allocator: O(1) alloc, O(1) reset. Thread-local scratch buffers for hot paths. Fields: `buffer []u8`, `offset usize`, `backing Allocator`.

Methods: `alloc(T, n)`, `create(T)`, `reset()`, `remaining()`, `used()`.

Used alongside `std.heap.ArenaAllocator` (stdlib) in the deb index parser.

---

### 3.19 Execution Primitives — `src/exec/`

#### `thread_pool.zig` — Chase-Lev work-stealing thread pool

- `Task` struct with function pointer + `cast()` downcast helper
- `TaskGroup` — atomic completion counter + `ResetEvent` for batch synchronization
- `WorkStealingDeque(T)` — Chase-Lev 2005 lock-free deque: owner pushes/pops bottom, thieves steal top
- `ThreadPool` — persistent worker threads parked on futex when idle

> Note: The thread pool is available in the library but `src/main.zig` uses `std.Thread.spawn` directly with a sliding window of 16 for the install pipeline. The pool is available for future use.

#### `dir_queue.zig` — Lock-free MPMC ring buffer

- `WorkQueue` — 4096-slot ring buffer with atomic write/read positions
- `Slot` — 512-byte path buffer with atomic `len` + `ready` flag
- `push(path)`, `pop()`, `markProcessed()`, `isDone()` — wait-free operations via spin on slot `ready` flag
- Used for distributing parallel download/extract work

---

### 3.20 Version — `src/version.zig`

- `compareVersions(a, b)` → `std.math.Order` (`.lt`, `.eq`, `.gt`)
- Splits on `.` and `_` (Homebrew uses `_` for rebuild suffix)
- Numeric segment comparison with lexicographic fallback for non-numeric segments
- 17 unit tests covering edge cases (trailing zeros, pre-release tags, rebuild suffixes)

---

### 3.21 Security Tests — `src/security_test.zig`

Adversarial test suite covering:
1. **Path traversal** — `../../../etc/passwd` variants rejected by `extract.isPathSafe()`
2. **Null bytes** — `Database.writeJsonEscaped()` escapes `\0` as `\u0000`
3. **JSON injection** — `"`, `\`, control chars in package names/versions escaped
4. **Version string fuzzing** — extreme inputs to `compareVersions` don't panic
5. **Placeholder injection** — crafted inputs to `replacePlaceholders` produce expected output

---

### 3.22 Cloudflare Worker — `worker/`

- Serves `https://nanobrew.trilok.ai/install` — the one-liner install script
- Platform-detects OS + architecture, downloads the appropriate release tarball from GitHub Releases, verifies SHA256, runs `nb init`
- Deployed via **Cloudflare Workers** (`wrangler.toml`, custom domain `nanobrew.trilok.ai`)
- `worker/src/index.js` — the Worker source (returns the install shell script as a response)

---

## 4. Dependency Graph

### Inter-module dependencies (internal)

```
main.zig
  ├─ root.zig (re-exports all modules as "nanobrew")
  ├─ api/client.zig
  │    ├─ api/formula.zig
  │    ├─ api/cask.zig
  │    ├─ api/tap.zig
  │    │    ├─ api/formula.zig
  │    │    ├─ api/cask.zig
  │    │    └─ net/fetch.zig
  │    ├─ net/fetch.zig
  │    └─ platform/paths.zig
  ├─ resolve/deps.zig
  │    ├─ api/client.zig
  │    └─ api/formula.zig
  ├─ net/downloader.zig
  │    ├─ store/store.zig
  │    └─ platform/paths.zig
  ├─ net/fetch.zig
  ├─ store/store.zig
  │    └─ extract/tar.zig
  │         └─ platform/paths.zig
  ├─ store/blob_cache.zig
  │    └─ platform/paths.zig
  ├─ cellar/cellar.zig
  │    ├─ platform/paths.zig
  │    └─ platform/copy.zig
  ├─ linker/linker.zig
  │    └─ platform/paths.zig
  ├─ platform/platform.zig
  │    ├─ platform/paths.zig
  │    ├─ platform/copy.zig
  │    ├─ platform/relocate.zig
  │    │    ├─ macho/relocate.zig  [macOS only]
  │    │    │    ├─ platform/paths.zig
  │    │    │    └─ platform/placeholder.zig
  │    │    ├─ elf/relocate.zig    [Linux only]
  │    │    │    ├─ platform/placeholder.zig
  │    │    │    └─ platform/paths.zig
  │    │    └─ platform/placeholder.zig
  │    └─ platform/placeholder.zig
  ├─ db/database.zig
  │    └─ platform/paths.zig
  ├─ cask/install.zig
  │    ├─ api/cask.zig
  │    ├─ platform/paths.zig
  │    └─ net/fetch.zig
  ├─ deb/distro.zig
  │    └─ platform/platform.zig
  ├─ deb/index.zig
  ├─ deb/resolver.zig
  │    └─ deb/index.zig
  ├─ deb/extract.zig
  │    ├─ platform/paths.zig
  │    └─ extract/native_tar.zig
  ├─ services/services.zig
  │    ├─ services/launchd.zig  [macOS]
  │    │    └─ platform/paths.zig
  │    └─ services/systemd.zig  [Linux]
  ├─ build/source.zig
  │    ├─ api/formula.zig
  │    ├─ net/fetch.zig
  │    └─ platform/paths.zig
  ├─ build/postinstall.zig
  │    ├─ api/formula.zig
  │    └─ net/fetch.zig
  ├─ api/search.zig
  │    ├─ net/fetch.zig
  │    └─ platform/paths.zig
  ├─ kernel/simd_scanner.zig    [standalone]
  ├─ kernel/mmap_reader.zig     [standalone]
  ├─ mem/arena.zig              [standalone]
  ├─ exec/thread_pool.zig       [standalone]
  ├─ exec/dir_queue.zig         [standalone]
  └─ version.zig                [standalone]
```

### External service dependencies

```
nanobrew binary
  ├─ formulae.brew.sh/api/         (Homebrew JSON API)
  ├─ ghcr.io/homebrew/core/        (bottle downloads, OCI registry)
  ├─ raw.githubusercontent.com/     (tap formula .rb files)
  ├─ archive.ubuntu.com/           (APT package index + .deb downloads)
  ├─ ports.ubuntu.com/             (APT arm64)
  ├─ deb.debian.org/               (APT for Debian)
  └─ api.github.com/               (nb update — release metadata)

Runtime system tools (optional, OS-provided):
  ├─ tar          (bottle extraction in extract/tar.zig)
  ├─ codesign     (macOS — batch re-sign after relocation)
  ├─ install_name_tool (macOS — fix dylib load paths)
  ├─ hdiutil      (macOS — mount .dmg casks)
  ├─ patchelf     (Linux — fix ELF RPATH/interpreter)
  ├─ cp           (COW copy fallback)
  └─ systemctl / launchctl (service management)
```

---

## 5. Install Pipeline Walkthrough

### Homebrew bottle (`nb install ffmpeg`)

```
1. RESOLVE  deps.zig:DepResolver.resolve("ffmpeg")
            └─ BFS: fetch ffmpeg JSON → 11 dep JSONs in parallel
            └─ topologicalSort() → install order [dep1, dep2, ..., ffmpeg]

2. FILTER   main.zig:runInstall
            └─ Skip packages with Cellar/<name>/<ver>/ already present
            └─ Heal DB drift for skipped packages

3. DOWNLOAD  [up to 16 parallel threads]
   per pkg: downloader.downloadOne(bottle_url, sha256)
            └─ Get GHCR Bearer token (4-min cache)
            └─ Stream HTTP → Sha256HashedReader → cache/tmp/<sha>
            └─ Rename to cache/blobs/<sha256>

4. EXTRACT   store.ensureEntry(blob_path, sha256)
            └─ tar xzf blob_path -C store/<sha256>/

5. MATERIALIZE  cellar.materialize(sha256, name, version)
               └─ clonefile(store/<sha>/<name>/<ver>/, Cellar/<name>/<ver>/)
               └─ Fallback: cp -R (or cp --reflink=auto on Linux)

6. RELOCATE  platform.relocate.relocateKeg(name, version)
   macOS:   └─ Parse Mach-O headers → find @@HOMEBREW_*@@ in load commands
            └─ install_name_tool per modified binary
            └─ codesign --force --sign - [all modified, one call]
   Linux:   └─ Parse ELF headers → patchelf --set-rpath per binary
            └─ Replace placeholders in .pc/.cmake/.la files

6b. PLACEHOLDER  replaceKegPlaceholders(name, version)
               └─ Walk all text files, replace @@HOMEBREW_*@@ strings

7. LINK   linker.linkKeg(name, version)
          └─ Cellar/<name>/<ver>/bin/* → prefix/bin/ (symlinks)
          └─ prefix/opt/<name> → keg root (symlink)

8. POST   postinstall.runPostInstall(formula)
          └─ Print caveats
          └─ Run post_install Ruby block (if defined)

9. RECORD  database.recordInstall(name, version, sha256)
```

### APT deb package (`nb install --deb curl`)

```
1. DETECT    deb/distro.detect()   → ubuntu noble amd64

2. INDEX     fetch Packages.gz from archive.ubuntu.com/{main,universe}
             └─ decompress gzip
             └─ deb/index.parsePackagesIndex() → 70K DebPackage (arena-alloc)

3. PROVIDES  build StringHashMap: virtual_name → real_package_name

4. RESOLVE   deb/resolver.resolveTransitiveDeps()
             └─ parseDependsField() per package (handles alternatives + virtuals)
             └─ topological sort → install order

5. DOWNLOAD  [8 threads]
             per pkg: fetch https://archive.ubuntu.com/<filename>
             └─ streaming SHA256 verification
             └─ save to cache/apt/<sha256>

6. EXTRACT   deb/extract.extractDebToPrefixWithFiles(deb_path)
             └─ Parse ar archive → find data.tar.*
             └─ Decompress (gzip/zstd/xz)
             └─ native_tar.extractToDir(data, "/")
             └─ Return file list

7. POSTINST  deb/extract.runPostinst(deb_path, name, skip_postinst)
             └─ Extract control.tar → run postinst script

8. RECORD    database.recordDebInstall(name, version, files, sha256)
```

---

## 6. Runtime Directory Layout

```
/opt/nanobrew/
├── cache/
│   ├── blobs/          # Downloaded bottle tarballs (named by SHA256)
│   ├── api/            # Cached formula JSON responses (5-min TTL)
│   │   ├── <name>.json         # Formula metadata
│   │   └── cask-<token>.json   # Cask metadata
│   │   └── _formula_list.json  # Full formula list (1-hr TTL, for search)
│   │   └── _cask_list.json     # Full cask list (1-hr TTL)
│   ├── tokens/         # GHCR bearer tokens (4-min TTL)
│   ├── apt/            # Downloaded .deb blobs (named by SHA256)
│   └── tmp/            # Partial downloads (atomically renamed on completion)
├── store/
│   └── <sha256>/       # Extracted bottle contents (one dir per unique bottle)
│       └── <name>/
│           └── <version>/
│               ├── bin/
│               ├── lib/
│               └── ...
├── prefix/
│   ├── Cellar/         # Installed packages (COW clones from store/)
│   │   └── <name>/
│   │       └── <version>/
│   ├── Caskroom/       # Installed macOS apps
│   │   └── <token>/
│   │       └── <version>/
│   ├── bin/            # Symlinks to binaries from Cellar
│   └── opt/            # Symlinks to keg roots
├── db/
│   └── state.json      # Installation state (kegs, casks, debs, history)
└── locks/              # Lock files (reserved, unused in current version)
```

---

## 7. Build System

`build.zig` defines the following build steps:

| Step | Command | Description |
|------|---------|-------------|
| default | `zig build` | Debug build → `zig-out/bin/nb` |
| release | `zig build -Doptimize=ReleaseFast` | Optimized build |
| run | `zig build run -- <args>` | Build + run |
| test | `zig build test` | All unit tests via `src/root.zig` |
| test-api | `zig build test-api` | API client tests only |
| test-tap | `zig build test-tap` | Tap formula parser tests |
| test-cask | `zig build test-cask` | Cask parser tests |
| test-deb-index | `zig build test-deb-index` | Deb index parser tests (7 tests) |
| test-deb-resolver | `zig build test-deb-resolver` | Deb resolver tests (17 tests) |
| test-deb-extract | `zig build test-deb-extract` | Deb extractor tests |
| test-deb-distro | `zig build test-deb-distro` | Distro detection tests |
| test-version | `zig build test-version` | Version comparison tests |
| test-tar | `zig build test-tar` | Native tar parser tests |
| test-security | `zig build test-security` | Security/adversarial tests |
| test-search | `zig build test-search` | Search API tests |
| linux | `zig build linux` | Cross-compile → `x86_64-linux-musl` |
| linux-arm | `zig build linux-arm` | Cross-compile → `aarch64-linux-musl` |

**Module structure in the build:**
- `nb_mod` — library module rooted at `src/root.zig`
- Main executable imports `nb_mod` as `"nanobrew"` 
- Tests are per-module (atomic — one crash doesn't kill others)
- Linux targets compile with `ReleaseFast` + `strip = true`

---

## 8. CI / CD

### `ci.yml` — Continuous Integration

| Job | Runner | Steps |
|-----|--------|-------|
| `build-macos` | `macos-14` | Debug + ReleaseFast build, `nb help`, `nb init`, unit tests, smoke tests |
| `build-linux` | `ubuntu-latest` | Cross-compile x86_64-linux, `nb help`, `nb init`, deb parity test (continue-on-error) |
| `cross-compile` | `macos-14` | Cross-compile x86_64-linux + aarch64-linux |

Zig version: **0.15.2** (pinned via `mlugg/setup-zig@v2`)

### `release.yml` — Release pipeline

Cross-compiles all 4 targets (arm64-macos, x86_64-macos, x86_64-linux, aarch64-linux), creates GitHub Release with SHA256 checksums.

### `benchmark.yml` — Weekly benchmarks

Runs on Apple Silicon (`macos-14`), compares nanobrew vs zerobrew vs Homebrew for a standard package set, updates `BENCHMARKS.md` in-repo.

---

## 9. Adoption Checklist

### Prerequisites

- [ ] **Zig 0.15+** — required to build from source (`zig build`)
- [ ] **macOS** (arm64 or x86_64) or **Linux** (aarch64 or x86_64)
- [ ] **`/opt/nanobrew/` writable** by the target user (run `sudo nb init` once at setup)
- [ ] **`tar`** available on `$PATH` (macOS: built-in; Linux: `apt install tar` or equivalent)
- [ ] **macOS only:** `codesign`, `install_name_tool`, `hdiutil` (Xcode Command Line Tools)
- [ ] **Linux only:** `patchelf` on `$PATH` for Homebrew bottle relocation (`apt install patchelf`)

### Installation

- [ ] One-liner: `curl -fsSL https://nanobrew.trilok.ai/install | bash`
- [ ] Or via Homebrew: `brew tap justrach/nanobrew && brew install nanobrew`
- [ ] Or build from source: `git clone && cd nanobrew && ./install.sh`
- [ ] Add `export PATH="/opt/nanobrew/prefix/bin:$PATH"` to shell profile

### First-run setup

- [ ] Run `nb init` (creates `/opt/nanobrew/` directory tree)
- [ ] Verify: `nb doctor` (checks permissions, PATH, tool availability)
- [ ] Optional: `nb migrate` to import existing Homebrew packages

### Homebrew migration

- [ ] Audit `brew bundle dump > Brewfile` before switching
- [ ] Run `nb migrate` to import existing Cellar/Caskroom entries
- [ ] Test `nb list` / `nb outdated` after migration
- [ ] nanobrew installs to `/opt/nanobrew/` — does **not** interfere with `/opt/homebrew/`

### Docker / CI adoption (Linux deb mode)

- [ ] Copy binary into image: `COPY --from=nanobrew/nb /nb /usr/local/bin/nb`
- [ ] Initialize: `RUN nb init`
- [ ] Replace `apt-get install`: `RUN nb install --deb curl wget git`
- [ ] Use `--skip-postinst` flag for packages where postinst scripts are irrelevant
- [ ] Verify distro detection works: `cat /etc/os-release` must have `ID=` and `VERSION_CODENAME=`

### Mirror / proxy configuration

- [ ] Set `NANOBREW_API_DOMAIN` to override `formulae.brew.sh` (formula metadata)
- [ ] Set `NANOBREW_BOTTLE_DOMAIN` to override GHCR bottle CDN
- [ ] Both also accept the `HOMEBREW_*` variants for drop-in Homebrew compatibility

### Known limitations to verify

- [ ] **Ruby `post_install` hooks** — only a subset of Ruby DSL is interpreted; test packages that rely on these (e.g. OpenSSL cert setup)
- [ ] **Build-from-source** — supported (cmake/autotools/meson/make auto-detected) but less tested than bottles
- [ ] **Brewfile compatibility** — `brew "pkg"` and `cask "pkg"` lines work; Ruby DSL conditionals and `mas` entries are ignored
- [ ] **`patchelf` on Linux** — must be installed for Homebrew bottle relocation; deb packages do not require it
- [ ] **Musl TLS bug** — known issue with `std.http.Client` + musl on Linux (tracked; workaround: use pre-built binary from GitHub Releases)

### Rollback / recovery

- [ ] `nb rollback <pkg>` — reverts to previous version (tracked in DB history)
- [ ] `nb cleanup --dry-run` — preview cache cleanup before running
- [ ] `nb nuke` — complete uninstall (removes `/opt/nanobrew/` entirely; requires `yes` confirmation)
- [ ] Homebrew is unaffected — remove nanobrew without affecting `/opt/homebrew/`

### Operational

- [ ] `nb pin <pkg>` — prevent upgrades for production-critical packages
- [ ] `nb bundle dump` → commit `Nanobrew` file to repo for reproducible environments
- [ ] `nb bundle install` — re-create environment from bundle file (returns instantly if already satisfied)
- [ ] `nb update` — explicit self-update (nanobrew never auto-updates during installs)
- [ ] Shell completions: `nb completions zsh >> ~/.zshrc`
