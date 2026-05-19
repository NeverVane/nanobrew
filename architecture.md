# nanobrew — Architecture Overview

> **Repo:** [NeverVane/nanobrew](https://github.com/NeverVane/nanobrew)  
> **Language:** Zig 0.15+  
> **Binary size:** ~1.2 MB static, no runtime dependencies  
> **Version surveyed:** 0.1.083 (2026-04-01)

---

## Table of Contents

1. [Project Summary](#1-project-summary)
2. [Repository Layout](#2-repository-layout)
3. [Module-by-Module Primer](#3-module-by-module-primer)
   - [Entry Point — `src/main.zig`](#31-entry-point--srcmainzig)
   - [Library Root — `src/root.zig`](#32-library-root--srcrootzig)
   - [API Layer — `src/api/`](#33-api-layer--srcapi)
   - [Dependency Resolution — `src/resolve/`](#34-dependency-resolution--srcresolve)
   - [Network Layer — `src/net/`](#35-network-layer--srcnet)
   - [Extraction Layer — `src/extract/`](#36-extraction-layer--srcextract)
   - [Content-Addressable Store — `src/store/`](#37-content-addressable-store--srcstore)
   - [Cellar — `src/cellar/`](#38-cellar--srccellar)
   - [Linker — `src/linker/`](#39-linker--srclinker)
   - [Platform Abstraction — `src/platform/`](#310-platform-abstraction--srcplatform)
   - [Binary Patching — `src/macho/` and `src/elf/`](#311-binary-patching--srcmacho-and-srcelf)
   - [Debian Package Support — `src/deb/`](#312-debian-package-support--srcdeb)
   - [macOS Cask Installer — `src/cask/`](#313-macos-cask-installer--srccask)
   - [Build-from-Source — `src/build/`](#314-build-from-source--srcbuild)
   - [Service Management — `src/services/`](#315-service-management--srcservices)
   - [Kernel Utilities — `src/kernel/`](#316-kernel-utilities--srckernel)
   - [Memory Utilities — `src/mem/`](#317-memory-utilities--srcmem)
   - [Concurrency Utilities — `src/exec/`](#318-concurrency-utilities--srcexec)
   - [Version Comparison — `src/version.zig`](#319-version-comparison--srcversionzig)
   - [State Database — `src/db/`](#320-state-database--srcdb)
   - [Security Tests — `src/security_test.zig`](#321-security-tests--srcsecurity_testzig)
4. [Runtime Directory Layout](#4-runtime-directory-layout)
5. [Install Pipeline — Step by Step](#5-install-pipeline--step-by-step)
   - [Homebrew Bottle (macOS / Linux)](#51-homebrew-bottle-macos--linux)
   - [macOS Cask](#52-macos-cask)
   - [Debian Package (Linux)](#53-debian-package-linux)
6. [Dependency Graph](#6-dependency-graph)
   - [Source-Level Module Graph](#61-source-level-module-graph)
   - [External / Runtime Dependencies](#62-external--runtime-dependencies)
7. [Concurrency Model](#7-concurrency-model)
8. [Performance Design Choices](#8-performance-design-choices)
9. [CI / CD and Release Pipeline](#9-ci--cd-and-release-pipeline)
10. [Cloudflare Worker (`worker/`)](#10-cloudflare-worker-worker)
11. [Homebrew Tap Formula (`Formula/`)](#11-homebrew-tap-formula-formula)
12. [Adoption Checklist](#12-adoption-checklist)

---

## 1. Project Summary

nanobrew is a **fast, drop-in Homebrew replacement** written entirely in Zig. It consumes the same Homebrew formula JSON API, the same pre-built bottle tarballs (from GitHub Container Registry), and the same cask definitions — but replaces the Ruby runtime with a single 1.2 MB static binary that starts instantly and parallelises every stage of installation.

Key differentiators over standard Homebrew:

| Capability | Implementation |
|---|---|
| No Ruby runtime | Single static Zig binary, no interpreter overhead |
| Parallel installs | Up to 16 concurrent download+extract+link threads |
| Content-addressable store | SHA256-keyed `/opt/nanobrew/store/` — warm installs skip all I/O |
| APFS clonefile / btrfs reflink | Zero-copy keg materialisation |
| Native HTTP (no curl) | `std.http.Client` with streaming SHA256, GHCR bearer token caching |
| Native binary patching | Reads Mach-O / ELF headers directly; no `otool` subprocess |
| Third-party tap support | Fetches and parses Ruby `.rb` formula files from GitHub |
| Native .deb support | Full APT replacement: index fetch → dep resolve → ar+gzip/zstd extract |

---

## 2. Repository Layout

```
nanobrew/
├── src/
│   ├── main.zig              # CLI entry point (all commands)
│   ├── root.zig              # Library module — re-exports all sub-modules
│   ├── api/
│   │   ├── client.zig        # Homebrew JSON API client + caching
│   │   ├── formula.zig       # Formula struct + bottle tag selection
│   │   ├── cask.zig          # Cask struct + artifact types
│   │   ├── search.zig        # Formula/cask search API
│   │   └── tap.zig           # Third-party tap Ruby parser
│   ├── resolve/
│   │   └── deps.zig          # BFS dep resolver + Kahn's topological sort
│   ├── net/
│   │   ├── downloader.zig    # Parallel bottle downloader + GHCR auth
│   │   └── fetch.zig         # Low-level HTTP GET helper
│   ├── extract/
│   │   ├── tar.zig           # gzip+tar extraction (store ingestion)
│   │   └── native_tar.zig    # Pure-Zig tar parser (for .deb data.tar)
│   ├── store/
│   │   ├── store.zig         # Content-addressable store API
│   │   └── blob_cache.zig    # Blob path helpers
│   ├── cellar/
│   │   └── cellar.zig        # COW copy from store into Cellar
│   ├── linker/
│   │   └── linker.zig        # bin/ + opt/ symlink management
│   ├── platform/
│   │   ├── platform.zig      # Comptime OS/arch detection hub
│   │   ├── paths.zig         # All path constants
│   │   ├── copy.zig          # clonefile / reflink / cp fallback
│   │   ├── relocate.zig      # Dispatches to macho/ or elf/ relocator
│   │   └── placeholder.zig   # @@HOMEBREW_*@@ text-file replacement
│   ├── macho/
│   │   └── relocate.zig      # Mach-O header parser + install_name_tool
│   ├── elf/
│   │   └── relocate.zig      # ELF header parser + patchelf
│   ├── deb/
│   │   ├── index.zig         # APT Packages index parser (ArenaAllocator)
│   │   ├── resolver.zig      # Deb dep resolver + virtual package support
│   │   ├── extract.zig       # .deb ar-archive + data.tar extractor
│   │   └── distro.zig        # /etc/os-release distro detection
│   ├── cask/
│   │   └── install.zig       # DMG/ZIP/PKG cask install pipeline
│   ├── build/
│   │   ├── source.zig        # Source-build pipeline (cmake/autotools/meson)
│   │   └── postinstall.zig   # Generic post-install hook runner
│   ├── services/
│   │   ├── services.zig      # OS-dispatch for service management
│   │   ├── launchd.zig       # macOS launchctl integration
│   │   └── systemd.zig       # Linux systemd integration
│   ├── kernel/
│   │   ├── simd_scanner.zig  # Comptime SIMD byte scanner (zigrep reuse)
│   │   └── mmap_reader.zig   # mmap-based zero-copy file reader
│   ├── mem/
│   │   └── arena.zig         # Arena allocator wrapper
│   ├── exec/
│   │   ├── thread_pool.zig   # Chase-Lev work-stealing thread pool
│   │   └── dir_queue.zig     # Directory BFS queue for parallel walks
│   ├── db/
│   │   └── database.zig      # JSON state database (kegs, casks, debs, history)
│   ├── version.zig           # Version string comparison
│   └── security_test.zig     # Input sanitisation / injection tests
├── worker/
│   └── src/index.js          # Cloudflare Worker (install script + landing page)
├── bench/                    # Docker-based benchmark harness
├── Formula/
│   └── nanobrew.rb           # Homebrew tap formula for self-distribution
├── tests/
│   ├── smoke-test.sh         # End-to-end install/remove smoke tests
│   └── deb-parity.sh         # apt-get vs nb --deb parity checks
├── .github/workflows/
│   ├── ci.yml                # Build + test + cross-compile on push/PR
│   ├── release.yml           # Builds release artifacts + GitHub Release
│   └── benchmark.yml         # Weekly automated benchmarks
├── build.zig                 # Zig build script (exe + test + cross targets)
├── install.sh                # Shell installer (detects arch, downloads release)
├── README.md
├── CHANGELOG.md
└── SECURITY.md
```

---

## 3. Module-by-Module Primer

### 3.1 Entry Point — `src/main.zig`

**Size:** 3 541 lines — the largest file.  
**Role:** Parses `argv`, dispatches to one handler per command, drives the full install pipeline.

**Commands handled:**

| Command | Handler | Description |
|---|---|---|
| `init` | `runInit` | Create `/opt/nanobrew/` directory tree |
| `install` | `runInstall` / `runCaskInstall` / `runDebInstall` | Multi-mode install |
| `remove` | `runRemove` | Unlink + delete keg/cask/deb |
| `reinstall` | remove then install | — |
| `list` | `runList` | Print kegs, casks, debs from DB |
| `leaves` | `runLeaves` | Show packages with no dependents |
| `info` | `runInfo` | Show formula / cask metadata |
| `search` | `runSearch` | Search Homebrew API |
| `upgrade` | `runUpgrade` | Parallel version check + re-install |
| `outdated` | `runOutdated` | Report available updates |
| `pin` / `unpin` | `runPin` | Toggle pinned flag in DB |
| `rollback` | `runRollback` | Restore previous version from history |
| `bundle` | `runBundle` | Dump/install Nanobrew bundle file |
| `deps` | `runDeps` | Print transitive dependency list |
| `services` | `runServices` | Start/stop launchd/systemd services |
| `completions` | `runCompletions` | Emit shell completions (zsh/bash/fish) |
| `doctor` | `runDoctor` | Sanity-check environment |
| `cleanup` | `runCleanup` | Remove stale blobs, old store entries |
| `update` | `runUpdate` | Self-update from GitHub Releases |
| `migrate` | `runMigrate` | Import packages from existing Homebrew install |
| `nuke` | `runNuke` | Completely remove nanobrew |

**Key internal types:**

```zig
const Phase = enum(u8) {
    waiting, downloading, extracting, installing, relocating, linking, done, failed
};
```

The `fullInstallOne` function runs the complete pipeline for a single package in its own thread:

```
download blob → extract to store → materialize to Cellar → relocate binary → link → post-install
```

A sliding-window scheduler (max 16 concurrent threads) prevents thread burst without artificial barriers.

**Security note:** Package names are validated by `isPackageNameSafe` before any filesystem access — rejecting `..`, null bytes, control characters, and names with path separators (unless the `user/tap/formula` pattern with exactly 2 slashes).

---

### 3.2 Library Root — `src/root.zig`

Zig module entry point that re-exports all sub-modules under stable public names (e.g. `pub const formula = @import("api/formula.zig")`). The `build.zig` builds this as the `nanobrew` library module imported by `main.zig` via `@import("nanobrew")`.

Also includes a comptime `_ = module;` stanza for each module to ensure the Zig test runner discovers tests in all sub-packages when `zig build test` is run.

---

### 3.3 API Layer — `src/api/`

#### `formula.zig` — Formula struct

Holds all data parsed from `formulae.brew.sh/api/formula/<name>.json`:

```
name, version, revision, rebuild, desc, dependencies[], build_deps[],
bottle_url, bottle_sha256, source_url, source_sha256, caveats, post_install_defined
```

`BOTTLE_TAG` and `BOTTLE_FALLBACKS` are `comptime` constants chosen at build time for the target OS/arch (e.g. `arm64_sonoma`, then `arm64_sequoia`, `arm64_ventura`, `all` as fallbacks).

#### `client.zig` — Homebrew API client

- Fetches formula JSON from `formulae.brew.sh/api/formula/<name>.json` (or cask variant).
- **Caching:** 1-hour TTL JSON files in `/opt/nanobrew/cache/api/`.
- **Custom domains:** Respects `NANOBREW_API_DOMAIN` / `HOMEBREW_API_DOMAIN` env vars for mirror support.
- **Tap routing:** If `name` contains exactly 2 slashes, delegates to `tap.fetchTapFormula`.
- Parses JSON using `std.json` (DOM-style) — extracts name, version, deps, bottle URLs, artifact lists.

#### `cask.zig` — Cask struct + artifact types

Holds `Cask` with `Artifact` union (`.app`, `.binary`, `.pkg`, `.uninstall`). Detects download format (DMG, ZIP, PKG, tar.gz) from URL extension.

#### `tap.zig` — Third-party tap Ruby parser (889 lines)

Fetches `.rb` formula/cask files directly from `raw.githubusercontent.com/<user>/homebrew-<tap>/HEAD/Formula/<name>.rb`. Falls back to several alternative URL patterns (sharded `Formula/n/name.rb`, repo-root placement).

Implements a line-by-line Ruby DSL parser that handles:
- `version`, `url`, `sha256`, `desc`, `homepage`, `name`
- `on_macos`/`on_linux`/`else`/`end` platform conditionals
- `if Hardware::CPU.intel?` / `arm?` architecture blocks
- `bottle do ... end` blocks with per-platform SHA256 and cellar fields
- `#{version}` string interpolation
- `depends_on "pkg"`, `:recommended`, `:optional` (optionals skipped)
- Cask artifacts: `app "Foo.app"`, `binary "src"`, `pkg "installer.pkg"`, `uninstall`

#### `search.zig` — Search API

Fetches `formulae.brew.sh/api/formula.json` and `formulae.brew.sh/api/cask.json` (full listings), filters client-side by substring match on name/description. Caches with 1-hour TTL.

---

### 3.4 Dependency Resolution — `src/resolve/deps.zig`

`DepResolver` implements **BFS with parallel level fetching + Kahn's algorithm topological sort**.

```
resolve("ffmpeg")
  → BFS level 0: fetch ffmpeg formula → discover [lame, opus, x265, ...]
  → BFS level 1: parallel-fetch all unknowns → discover transitive deps
  → BFS level N: until frontier is empty
  → topologicalSort() → Kahn's algorithm with self-edge skip + cycle detection
```

Key properties:
- Shares one `std.http.Client` across single-item fetches (TLS connection reuse).
- Multi-item levels spawn one thread per item (no shared client — each thread creates its own).
- Missing dependency detected before Kahn's sort (returns `MissingDependency` not `DependencyCycle`).
- Self-dependencies (packages like `r`, `neomutt`) correctly skipped.

---

### 3.5 Network Layer — `src/net/`

#### `downloader.zig` — Parallel bottle downloader (363 lines)

`downloadOne(req)`:
1. Rewrites URL if `NANOBREW_BOTTLE_DOMAIN` / `HOMEBREW_BOTTLE_DOMAIN` set.
2. Fetches GHCR bearer token from `https://ghcr.io/token?scope=repository:<repo>:pull` — token cached 4 min in `/opt/nanobrew/cache/tokens/`.
3. Downloads with native `std.http.Client` (5 redirects allowed).
4. Streams response body through a `hashed` reader — computes SHA256 in a single pass.
5. Writes to `cache/tmp/<sha256>.dl`, then renames atomically to `cache/blobs/<sha256>`.
6. Verifies SHA256 before rename; deletes tmp file on mismatch.

`ParallelDownloader` queues requests and dispatches up to 8 worker threads (lock-free atomic index).

#### `fetch.zig` — HTTP GET helper

Simple `fetch.get(alloc, url)` returning `[]u8`. Used for small JSON payloads (API, token responses, Ruby formula files). Also provides `getWithClient` for shared-client reuse.

---

### 3.6 Extraction Layer — `src/extract/`

#### `tar.zig` — Bottle extraction

`extractToStore(alloc, blob_path, sha256)`: extracts a gzip-compressed bottle tarball from `cache/blobs/<sha256>` into `store/<sha256>/`. Uses standard `std.compress.gzip` + ustar header parsing.

#### `native_tar.zig` — Pure-Zig tar parser (499 lines)

`extractToDir(alloc, data, dest_dir)`: extracts an in-memory tar byte slice to a directory. Returns `[][]const u8` — list of all extracted file paths (used for deb file tracking).

Supports: regular files, symlinks, hardlinks, directories. Handles GNU long-name headers. Validates path segments to prevent directory traversal.

---

### 3.7 Content-Addressable Store — `src/store/`

#### `store.zig`

Simple API over `/opt/nanobrew/store/<sha256>/`:
- `hasEntry(sha256)` — O(1) filesystem access check.
- `ensureEntry(alloc, blob_path, sha256)` — extracts blob if entry missing.
- `removeEntry(sha256)` — recursive delete (used by `cleanup`).

The store is the deduplication layer: if two formulas share a dependency with the same SHA256, only one copy is stored.

#### `blob_cache.zig`

Path construction helpers for `cache/blobs/`.

---

### 3.8 Cellar — `src/cellar/cellar.zig`

`materialize(sha256, name, version)`:
1. Locates `store/<sha256>/<name>/<version>/` (Homebrew nested layout).
2. Handles version fuzzy-match (e.g. API returns `3.1.0` but bottle dir is `3.1.0_1`).
3. Attempts **`clonefile(2)`** (macOS APFS COW) via `platform/copy.zig`.
4. Falls back to `cp --reflink=auto` (btrfs/xfs) then plain recursive copy.
5. Destination: `prefix/Cellar/<name>/<version>/`.

Also provides `detectKegVersion` (filesystem scan for version suffixed with `_rebuild`) and `remove` (delete keg dir, prune empty parent).

---

### 3.9 Linker — `src/linker/linker.zig`

`linkKeg(name, version)`:
- Iterates `Cellar/<name>/<version>/bin/` and `sbin/` — creates symlinks in `prefix/bin/`.
- Creates `prefix/opt/<name>` → `Cellar/<name>/<version>` symlink (Homebrew convention).

`unlinkKeg(name, version)`:
- Verifies symlink target before removing (safety: only removes links that point to this keg).

---

### 3.10 Platform Abstraction — `src/platform/`

#### `platform.zig`

Comptime constants: `is_linux`, `is_macos`, `deb_arch` (`"arm64"` or `"amd64"`). Re-exports `paths`, `copy`, `relocate`, `placeholder`.

#### `paths.zig`

All filesystem paths as compile-time constants:

| Constant | Value |
|---|---|
| `ROOT` | `/opt/nanobrew` |
| `PREFIX` | `/opt/nanobrew/prefix` |
| `CELLAR_DIR` | `/opt/nanobrew/prefix/Cellar` |
| `BIN_DIR` | `/opt/nanobrew/prefix/bin` |
| `STORE_DIR` | `/opt/nanobrew/store` |
| `BLOBS_DIR` | `/opt/nanobrew/cache/blobs` |
| `DB_PATH` | `/opt/nanobrew/db/state.json` |
| `CASKROOM_DIR` | `/opt/nanobrew/prefix/Caskroom` |

#### `copy.zig`

`cloneTree(&src_z, &dst_z)` — calls `clonefile(2)` on macOS (returns bool success).  
`cpFallback(src, dest)` — spawns `cp --reflink=auto -R` on Linux, `cp -R` on macOS.

#### `relocate.zig`

OS-dispatch shim: calls `macho/relocate.relocateKeg` on macOS, `elf/relocate.relocateKeg` on Linux. Also calls `placeholder.replaceKegPlaceholders` on both.

#### `placeholder.zig` (425 lines)

Scans text files in a keg for `@@HOMEBREW_PREFIX@@`, `@@HOMEBREW_CELLAR@@`, `@@HOMEBREW_REPOSITORY@@`, `@@HOMEBREW_LIBRARY@@` and replaces them with nanobrew's actual paths. Uses mmap + SIMD scanner for performance on large files.

---

### 3.11 Binary Patching — `src/macho/` and `src/elf/`

#### `macho/relocate.zig` (385 lines)

1. Walks `bin/`, `sbin/`, `lib/`, `libexec/`, `Frameworks/` in the keg.
2. Reads first 4 bytes of each file — checks for `0xFEEDFACF` (Mach-O 64), `0xCAFEBABE` (Fat binary).
3. Parses load commands natively (no `otool` subprocess) — looks for `LC_ID_DYLIB`, `LC_LOAD_DYLIB`, `LC_RPATH` containing `@@HOMEBREW_PREFIX@@` or `@@HOMEBREW_CELLAR@@`.
4. If placeholder found: spawns **one `install_name_tool` call** per binary.
5. After all binaries in keg: **one batch `codesign --sign - --force`** call for all modified files.

**Old approach:** 3N subprocess spawns (otool + install_name_tool + codesign per binary).  
**New approach:** N+1 spawns (install_name_tool per modified binary + 1 codesign batch).

#### `elf/relocate.zig` (269 lines)

Mirrors Mach-O relocator for Linux:
1. Detects ELF magic (`0x7f ELF`).
2. Checks ELF string table for HOMEBREW placeholders.
3. Spawns `patchelf --set-rpath` when placeholders found.
4. Replaces placeholders in `.pc`, `.cmake`, `.la`, `.sh`, `.cfg` text files.
5. No codesign step.

Emits a helpful error when `patchelf` is not installed.

---

### 3.12 Debian Package Support — `src/deb/`

#### `distro.zig` — Distro detection

Parses `/etc/os-release` → extracts `ID` and `VERSION_CODENAME`. Maps to APT mirror:
- Ubuntu amd64 → `http://archive.ubuntu.com/ubuntu`
- Ubuntu arm64 → `http://ports.ubuntu.com/ubuntu-ports`
- Debian → `http://deb.debian.org/debian`

Components: Ubuntu (`main`, `universe`) / Debian (`main`, `contrib`).

#### `index.zig` — APT Packages index parser (402 lines)

`parsePackagesIndex(alloc, data)` → `ParsedIndex` with an internal `ArenaAllocator`.

Splits on `\n\n` paragraph separators, extracts per-block fields: `Package`, `Version`, `Depends`, `Provides`, `Filename`, `SHA256`, `Size`, `Description`. The ArenaAllocator means a single `deinit()` frees all 70 K+ parsed packages.

#### `resolver.zig` — Deb dependency resolver (444 lines)

`parseDependsField` handles Debian `Depends:` syntax:
- Comma-separated groups
- `pkg1 | pkg2` alternatives (picks first present in index or provides map)
- `pkg (>= 1.0)` version constraints (stored but not enforced in v0)
- `:arch` qualifiers stripped

`resolveAll` implements topological resolution:
1. Builds `StringHashMap` index from parsed packages.
2. Builds `provides_map` for virtual packages (`build-essential` → `gcc`, etc.).
3. BFS from requested packages, resolving via index + provides.
4. Returns packages in install order (leaves first).

#### `extract.zig` — .deb extractor (372 lines)

A `.deb` is an `ar(1)` archive. This module:
1. Reads the 8-byte `!<arch>\n` magic.
2. Parses 60-byte ar member headers to find `data.tar.*`.
3. Detects compression: zstd, gzip, xz (xz falls back to subprocess).
4. Decompresses in memory.
5. Calls `native_tar.extractToDir` → returns file list for DB tracking.
6. Separately extracts `control.tar.*` and runs `postinst` scripts.

---

### 3.13 macOS Cask Installer — `src/cask/install.zig` (399 lines)

`installCask(alloc, cask)`:
1. Downloads artifact (DMG, ZIP, PKG, tar.gz) to `cache/tmp/`.
2. Creates `prefix/Caskroom/<token>/<version>/` directory.
3. **DMG:** Strips `com.apple.quarantine` xattr, `hdiutil attach`, copies `.app` bundles to `/Applications/`, `hdiutil detach`.
4. **ZIP:** Unzips to temp dir, finds `.app`, copies to `/Applications/`.
5. **PKG:** Runs `installer -pkg ... -target /`.
6. Creates `prefix/bin/` symlinks for `binary` artifacts.
7. Validates `.app` bundle exists before claiming success.

`removeCask`:
- Removes `.app` from `/Applications/`.
- Removes `prefix/bin/` symlinks.
- Removes `prefix/Caskroom/<token>/`.

---

### 3.14 Build-from-Source — `src/build/`

#### `source.zig` (242 lines)

Used when a formula has no bottle (bottle_url is empty) but has `source_url`. Pipeline:
1. Download tarball → verify SHA256 (via `shasum` subprocess).
2. Extract to temp dir.
3. Detect build system: looks for `CMakeLists.txt`, `configure`, `meson.build`, `Makefile`.
4. Run cmake/autotools/meson/make with install prefix set to keg dir.
5. Falls through to linker + relocator like a normal bottle install.

#### `postinstall.zig` (250 lines)

Runs non-Ruby post-install steps after a bottle is materialized:
- Runs `ldconfig` if shared libraries were installed (Linux).
- Runs `update-ca-certificates` for ca-certificates package.
- Sets up shell environment for packages that need it (e.g. Go, Node).

---

### 3.15 Service Management — `src/services/`

`services.zig` dispatches at comptime to `launchd.zig` (macOS) or `systemd.zig` (Linux).

Both implement the same interface: `Service` struct, `discoverServices`, `isRunning`, `start`, `stop`.

- **launchd:** Scans `Cellar/<pkg>/*/Library/LaunchDaemons/` and `LaunchAgents/` for plists. Uses `launchctl load/unload/list`.
- **systemd:** Scans `Cellar/<pkg>/*/lib/systemd/system/` for `.service` files. Uses `systemctl enable/disable/start/stop`.

---

### 3.16 Kernel Utilities — `src/kernel/`

#### `simd_scanner.zig` (315 lines) — Ported from zigrep

`ByteScanner(comptime simd_w)` generates a specialized scanner at compile time:
- `findFirst(haystack, needle)` — SIMD memchr with software prefetch.
- `scanAll(haystack, needle, callback)` — all-occurrences with first+last byte filter (Mula's technique).
- Auto-detects SIMD width: AVX-512 (64B), AVX2 (32B), SSE2/NEON (16B).

Used by `placeholder.zig` to locate `@@HOMEBREW_*@@` markers in large files without scanning byte-by-byte.

#### `mmap_reader.zig` (133 lines) — Ported from zigrep

Zero-copy file reading via `mmap`. Exposed as a standard reader interface. Used for scanning large binaries during placeholder replacement without loading them into heap.

---

### 3.17 Memory Utilities — `src/mem/arena.zig`

Thin wrapper around `std.heap.ArenaAllocator`. The deb index parser uses this to allocate all 70 K+ package string fields from a single arena, then free them in O(1) with `deinit()`.

---

### 3.18 Concurrency Utilities — `src/exec/`

#### `thread_pool.zig` (265 lines) — Ported from zigrep

**Chase-Lev work-stealing deque** (Chase & Lev, 2005):
- Per-worker `WorkStealingDeque` — owner pushes/pops from bottom, thieves steal from top.
- Workers park on `std.Thread.ResetEvent` when idle (futex-based).
- `TaskGroup` with atomic countdown + event for batch synchronisation.

Used for the deb parallel download+extract pipeline. The main install pipeline uses simpler ad-hoc thread spawning (sliding window, up to 16 threads).

#### `dir_queue.zig` (87 lines)

Thread-safe FIFO queue of directory paths for parallel tree walking (used in placeholder replacement and binary scanning).

---

### 3.19 Version Comparison — `src/version.zig` (150 lines)

`compareVersions(a, b)` splits on `.` and `_` (Homebrew uses `_` as rebuild suffix separator), compares segments numerically with string fallback. Handles asymmetric lengths (e.g. `10.47` vs `10.47_1`).

`isNewer(candidate, installed)` returns true if `compareVersions(candidate, installed) == .gt`.

Has 15+ unit tests covering edge cases: pre-release strings, rebuild suffixes, alpha versions.

---

### 3.20 State Database — `src/db/database.zig` (579 lines)

**File:** `/opt/nanobrew/db/state.json`  
**Format:** Hand-serialised JSON (no external JSON library for writing — uses `writeJsonEscaped` to prevent injection).

Tracks four collections:
- `kegs[]` — `{name, version, sha256, pinned, installed_at}`
- `casks[]` — `{token, version, apps[], binaries[]}`
- `history{}` — per-package array of `{version, sha256, installed_at}` (for `rollback`)
- `deb_packages[]` — `{name, version, sha256, installed_at, files[]}` (for clean removal)

`open(alloc)` → deserialises via `std.json`; tolerates missing keys for backward compatibility.  
`close()` → calls `save()`.  
`save()` → writes atomically with `file.sync()` before close.

All writes are hand-serialised with `writeJsonEscaped` which escapes `"`, `\`, newlines, and all control characters — preventing JSON injection from malicious package names.

---

### 3.21 Security Tests — `src/security_test.zig` (316 lines)

Exercises:
- `isPackageNameSafe` — path traversal, null bytes, control chars, too-long names, tap refs.
- `Database.writeJsonEscaped` — double quotes, backslashes, newlines, control chars, JSON injection payloads.
- `SUDO_USER` validation in `runInit`.

---

## 4. Runtime Directory Layout

```
/opt/nanobrew/
├── cache/
│   ├── blobs/          # Downloaded bottle tarballs (SHA256-named, immutable)
│   ├── api/            # Cached formula JSON (1-hour TTL)
│   ├── tokens/         # GHCR bearer tokens (4-minute TTL)
│   ├── apt/            # Cached APT package index (NBIX binary format)
│   └── tmp/            # In-progress downloads (atomic rename on completion)
├── store/
│   └── <sha256>/       # Extracted bottle contents (content-addressable)
│       └── <name>/
│           └── <version>/
│               ├── bin/
│               ├── lib/
│               └── ...
├── prefix/
│   ├── Cellar/
│   │   └── <name>/
│   │       └── <version>/   # COW copy from store/
│   ├── Caskroom/
│   │   └── <token>/
│   │       └── <version>/
│   ├── bin/            # Symlinks to Cellar binaries
│   └── opt/            # Symlinks to keg root dirs
└── db/
    └── state.json      # Installation state
```

---

## 5. Install Pipeline — Step by Step

### 5.1 Homebrew Bottle (macOS / Linux)

```
nb install ffmpeg
│
├─ 1. Parse args → validate package names (isPackageNameSafe)
│
├─ 2. DepResolver.resolve("ffmpeg")
│     ├─ BFS: fetch ffmpeg formula (JSON API / 1h cache)
│     ├─ Parallel-fetch all unknown dependencies
│     └─ Topological sort (Kahn's algorithm)
│
├─ 3. Filter already-installed packages
│     └─ For already-present kegs: heal relocations + re-link (idempotent)
│
├─ 4. Parallel install loop (≤16 concurrent threads, sliding window)
│     For each formula:
│     ├─ a. Download: downloadOne() → GHCR auth → streaming SHA256 → cache/blobs/<sha>
│     ├─ b. Extract:  store.ensureEntry() → gzip+tar → store/<sha>/
│     ├─ c. Materialize: cellar.materialize() → clonefile/reflink → Cellar/<name>/<ver>/
│     ├─ d. Relocate:
│     │      macOS: parse Mach-O headers → install_name_tool + batch codesign
│     │      Linux: parse ELF headers → patchelf --set-rpath
│     │      Both:  replace @@HOMEBREW_*@@ in text files
│     ├─ e. Link: symlinks in prefix/bin/ and prefix/opt/
│     └─ f. Post-install: ldconfig / ca-certificates / etc.
│
└─ 5. Record in DB (serial — single JSON file)
```

**Warm path:** Steps (a), (b), (c) are all skipped if blob/store/keg already exist. Warm install is dominated by symlink creation: ~3.5 ms for a single package.

### 5.2 macOS Cask

```
nb install --cask firefox
│
├─ 1. Fetch cask JSON (formulae.brew.sh/api/cask/firefox.json)
├─ 2. Download artifact (DMG/ZIP/PKG) to cache/tmp/
├─ 3. Strip com.apple.quarantine xattr from DMG
├─ 4. hdiutil attach → find .app → cp -R to /Applications/ → hdiutil detach
├─ 5. Symlink binary artifacts in prefix/bin/
├─ 6. Create Caskroom entry
└─ 7. Record in DB
```

### 5.3 Debian Package (Linux)

```
nb install --deb curl wget git
│
├─ 1. Detect distro: parse /etc/os-release → codename + mirror
├─ 2. Fetch APT index: GET <mirror>/dists/<codename>/<component>/binary-<arch>/Packages.gz
│     (main + universe, decompressed, parsed with ArenaAllocator)
├─ 3. Build provides_map (virtual package resolution)
├─ 4. Resolve deps: topological BFS via StringHashMap index
├─ 5. Download .debs: parallel (8 threads), streaming SHA256
├─ 6. Extract: parse ar archive → decompress data.tar.{zst,gz} → native_tar → /
├─ 7. Run postinst scripts (ca-certificates, ldconfig, etc.)
├─ 8. Run ldconfig for shared library registration
└─ 9. Record in DB with file list (for clean removal)
```

---

## 6. Dependency Graph

### 6.1 Source-Level Module Graph

```
main.zig
  └─ nanobrew (root.zig)
       ├─ api/client.zig
       │    ├─ api/formula.zig
       │    ├─ api/cask.zig
       │    ├─ api/tap.zig         → api/formula.zig, api/cask.zig, net/fetch.zig
       │    └─ net/fetch.zig
       │
       ├─ api/search.zig           → net/fetch.zig
       │
       ├─ resolve/deps.zig         → api/client.zig, api/formula.zig
       │
       ├─ net/downloader.zig       → store/store.zig, platform/paths.zig
       ├─ net/fetch.zig            (no internal deps)
       │
       ├─ extract/tar.zig          (no internal deps)
       ├─ extract/native_tar.zig   (no internal deps)
       │
       ├─ store/store.zig          → extract/tar.zig, platform/paths.zig
       ├─ store/blob_cache.zig     → platform/paths.zig
       │
       ├─ cellar/cellar.zig        → platform/paths.zig, platform/copy.zig
       │
       ├─ linker/linker.zig        → platform/paths.zig
       │
       ├─ platform/platform.zig
       │    ├─ platform/paths.zig  (no internal deps)
       │    ├─ platform/copy.zig   (syscalls only)
       │    ├─ platform/relocate.zig → macho/relocate.zig OR elf/relocate.zig
       │    └─ platform/placeholder.zig → kernel/simd_scanner.zig, kernel/mmap_reader.zig
       │
       ├─ macho/relocate.zig       → platform/paths.zig, platform/placeholder.zig
       ├─ elf/relocate.zig         → platform/placeholder.zig, platform/paths.zig
       │
       ├─ cask/install.zig         → api/cask.zig, platform/paths.zig, net/fetch.zig
       │
       ├─ build/source.zig         → api/formula.zig, net/fetch.zig, platform/paths.zig
       ├─ build/postinstall.zig    → api/formula.zig, platform/paths.zig
       │
       ├─ deb/index.zig            (no internal deps)
       ├─ deb/resolver.zig         → deb/index.zig
       ├─ deb/extract.zig          → extract/native_tar.zig, platform/paths.zig
       ├─ deb/distro.zig           → platform/platform.zig
       │
       ├─ services/services.zig    → services/launchd.zig OR services/systemd.zig
       │
       ├─ db/database.zig          → platform/paths.zig
       ├─ version.zig              (no internal deps)
       │
       ├─ kernel/simd_scanner.zig  (no internal deps)
       ├─ kernel/mmap_reader.zig   (no internal deps)
       ├─ mem/arena.zig            (no internal deps)
       ├─ exec/thread_pool.zig     (no internal deps)
       ├─ exec/dir_queue.zig       (no internal deps)
       │
       └─ security_test.zig        → db/database.zig (writeJsonEscaped)
```

**Dependency rules (no cycles):**
- `platform/` modules only depend on each other and `kernel/` — never upward.
- `api/` depends on `net/fetch.zig` only — not on `store/`, `cellar/`, or `db/`.
- `deb/` only depends on `extract/native_tar.zig` and `platform/` — not on brew API.
- `db/` has no dependency on any install-pipeline module — it is a pure data layer.

### 6.2 External / Runtime Dependencies

| Dependency | When required | Notes |
|---|---|---|
| `formulae.brew.sh` | Formula / cask install | Homebrew JSON API. Supports custom domain via `NANOBREW_API_DOMAIN`. |
| `ghcr.io` (GitHub Container Registry) | Bottle download | GHCR bearer token fetched automatically. Supports `NANOBREW_BOTTLE_DOMAIN` override. |
| `raw.githubusercontent.com` | Third-party tap installs | Ruby formula/cask files |
| `archive.ubuntu.com` / `ports.ubuntu.com` | `--deb` mode | APT package index + `.deb` files |
| `install_name_tool` | macOS bottle install | Part of Xcode CLI tools — always present on macOS |
| `codesign` | macOS bottle install | Part of Xcode CLI tools |
| `patchelf` | Linux bottle install with ELF placeholders | Must be installed separately (`apt-get install patchelf`). nanobrew warns clearly if missing. |
| `hdiutil` | macOS cask DMG install | Built into macOS |
| `installer` | macOS cask PKG install | Built into macOS |
| `cmake` / `autoconf` / `meson` / `make` | Source builds only | Only needed when no bottle exists |
| `tar` | `.deb` xz extraction fallback | Only for xz-compressed data.tar (rare); gzip/zstd handled natively |

**Zero external dependencies at runtime** for the common case (bottle install on macOS or deb install on Linux).

---

## 7. Concurrency Model

nanobrew uses **ad-hoc `std.Thread.spawn`** for most parallelism, not the Chase-Lev thread pool (which is present in the codebase but inherited from zigrep for search workloads).

| Operation | Parallelism |
|---|---|
| Dependency resolution (BFS levels) | One thread per unknown dep per BFS level |
| Bottle install pipeline | Sliding window of ≤16 threads; each thread owns its full pipeline (download → extract → materialize → relocate → link) |
| Parallel downloader | Up to 8 threads with atomic work-stealing index |
| Outdated / upgrade version check | Up to 8 threads |
| Progress display | Main thread polls `std::atomic.Value(u8)` phase array at 80 ms intervals |

All shared state across threads is accessed via `std.atomic.Value` — no mutexes in the hot path. The database is written serially after all threads complete.

---

## 8. Performance Design Choices

| Design | Rationale |
|---|---|
| Content-addressable store keyed by SHA256 | Same package version is extracted only once. Reinstall = symlink recreation only (~3.5 ms). |
| APFS `clonefile(2)` / btrfs `reflink` | Zero disk I/O and zero space for keg materialisation on supported filesystems. |
| Streaming SHA256 in download | Single pass over downloaded bytes; no re-read for verification. |
| Mach-O header parsed natively | Avoids spawning `otool` per binary. One codesign batch call instead of N. |
| ArenaAllocator for APT index | 70 K+ packages freed in O(1); zero fragmentation during parse. |
| SIMD byte scanner for placeholder search | Finds `@@HOMEBREW_*@@` patterns in large binaries 4–8× faster than scalar search. |
| GHCR token cache (4 min) | Avoids repeated token fetches across parallel downloads. |
| API cache (1 hour) | Avoids repeated HTTPS round-trips for dependency resolution on warm installs. |
| No auto-update on install | `nb install` never fetches nanobrew updates. Self-update is explicit (`nb update`). |
| Sliding window (≤16 threads) | Prevents burst of 16 threads stalling, then 16 more — keeps throughput smooth. |

---

## 9. CI / CD and Release Pipeline

### `ci.yml` — Runs on push/PR to `main`

1. **`build-macos`** (macos-14 / Apple Silicon)
   - `zig build` (Debug)
   - `zig build -Doptimize=ReleaseFast`
   - `zig build test` (all unit tests)
   - `bash tests/smoke-test.sh` (install + remove real packages)

2. **`build-linux`** (ubuntu-latest)
   - `zig build linux` (cross-compile x86_64-linux-musl)
   - `bash tests/deb-parity.sh` (apt-get vs nb --deb parity)

3. **`cross-compile`** (macos-14)
   - `zig build linux` (x86_64)
   - `zig build linux-arm` (aarch64)

### `release.yml` — Triggered on version tag

Builds 4 release artifacts:
- `nb-arm64-apple-darwin.tar.gz` + SHA256
- `nb-x86_64-apple-darwin.tar.gz` + SHA256
- `nb-aarch64-linux.tar.gz` + SHA256
- `nb-x86_64-linux.tar.gz` + SHA256

Publishes to GitHub Releases. The `nb update` self-update command fetches these.

### `benchmark.yml` — Weekly automated

Runs the Docker-based benchmark harness in `bench/` on macos-14 (Apple Silicon). Results are compared against stored baselines and committed back to `BENCHMARKS.md`.

---

## 10. Cloudflare Worker (`worker/`)

**Stack:** Cloudflare Workers (JavaScript, Bun runtime, Wrangler deploy)

`worker/src/index.js` serves:
- `GET /install` — streams the install shell script (content defined inline)
- `GET /` — landing page / redirect

The worker handles `https://nanobrew.trilok.ai/install` — the one-liner install URL in the README (`curl -fsSL https://nanobrew.trilok.ai/install | bash`).

`worker/wrangler.toml` configures the Cloudflare Worker name and routes.

---

## 11. Homebrew Tap Formula (`Formula/`)

`Formula/nanobrew.rb` is a standard Homebrew formula that installs nanobrew via Homebrew's bottle mechanism. This allows:

```bash
brew tap justrach/nanobrew https://github.com/justrach/nanobrew
brew install nanobrew
```

The formula is self-referential: it distributes the nanobrew binary via Homebrew so users who already have Homebrew can bootstrap without curl.

---

## 12. Adoption Checklist

Use this checklist when evaluating nanobrew for a new environment or project.

### Prerequisites

- [ ] **Zig 0.15+** — required to build from source (CI uses 0.15.2).
- [ ] **macOS:** Xcode CLI tools installed (`xcode-select --install`) — provides `install_name_tool`, `codesign`, `hdiutil`.
- [ ] **Linux (Homebrew bottles):** `patchelf` installed (`apt-get install patchelf`) — required only if installed packages embed ELF RPATH placeholders.
- [ ] **Linux (deb mode):** No external dependencies. Runs fully self-contained.

### Initial Setup

- [ ] Run `sudo nb init` — creates `/opt/nanobrew/` directory tree and chowns to real user.
- [ ] Add `export PATH="/opt/nanobrew/prefix/bin:$PATH"` to shell profile.
- [ ] Verify: `nb doctor` — checks path, permissions, and binary health.

### Migration from Homebrew

- [ ] Run `nb migrate` — scans `/opt/homebrew/Cellar/` (macOS) or `/home/linuxbrew/.linuxbrew/` (Linux) and imports existing packages into nanobrew's DB.
- [ ] Note: migrated packages will appear in `nb list` / `nb outdated` but are not re-installed to nanobrew's Cellar. Run `nb upgrade` to migrate binaries under nanobrew management.
- [ ] Packages from both Homebrew and nanobrew coexist safely — different install roots.

### Feature Compatibility

- [ ] Confirm all required formulae have Homebrew bottles (check `formulae.brew.sh`). Formulae that are source-only require build toolchain.
- [ ] Check for `post_install` hooks: `nb info <pkg>` shows `post_install_defined: true` when a formula has Ruby post-install logic that nanobrew cannot run. Most bottles are unaffected.
- [ ] Third-party taps: test `nb install user/tap/formula` — Ruby DSL edge cases may need fixes for unusual formula layouts.
- [ ] macOS casks with `auto_updates` flag: nanobrew won't auto-update these apps (consistent with its no-auto-update philosophy).
- [ ] `nb bundle install` supports `brew "pkg"` and `cask "pkg"` lines; `tap`, `mas`, and conditional Ruby blocks are ignored.

### Docker / Linux Container Adoption

- [ ] Use `COPY --from=nanobrew/nb /nb /usr/local/bin/nb` in Dockerfile to get the binary.
- [ ] Run `nb init` before any `nb install --deb` commands.
- [ ] Pass `--skip-postinst` flag if post-install scripts cause issues in your container environment.
- [ ] Consider `--no-verify` only if pulling from a trusted private mirror with known-good packages.

### Environment Variables

| Variable | Purpose |
|---|---|
| `NANOBREW_API_DOMAIN` | Override formulae.brew.sh for formula/cask JSON |
| `NANOBREW_BOTTLE_DOMAIN` | Override ghcr.io for bottle downloads (proxy / regional mirror) |
| `HOMEBREW_API_DOMAIN` | Alias for `NANOBREW_API_DOMAIN` (Homebrew compat) |
| `HOMEBREW_BOTTLE_DOMAIN` | Alias for `NANOBREW_BOTTLE_DOMAIN` (Homebrew compat) |

### Testing the Installation

```bash
nb install tree            # Simple 0-dep formula (~9ms warm)
nb install ffmpeg          # Multi-dep formula (~287ms warm)
nb list                    # Verify recorded in DB
nb remove tree             # Clean removal
nb doctor                  # Health check
zig build test             # Run all unit tests (dev only)
bash tests/smoke-test.sh ./zig-out/bin/nb   # Integration smoke tests (dev only)
```

### Known Limitations

- **Ruby `post_install` hooks** — not executed. Affects a small fraction of formulae.
- **`brew tap` as a standalone command** — not implemented; tap refs work inline (`nb install user/tap/formula`).
- **Mac App Store (`mas`)** — not supported in bundle files.
- **`--with-*` build options** — source builds use default configuration only.
- **Complex Brewfile Ruby DSL** — conditional blocks and custom Ruby code are ignored.
- **Linux ELF placeholders** — requires `patchelf`; will warn and skip relocation if absent.
- **macOS casks (Linux)** — cask install is macOS-only; returns error on Linux.
- **Experimental status** — project is self-described as experimental; test with non-critical packages first.
