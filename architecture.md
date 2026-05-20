# nanobrew — Architecture Overview

> **Repo:** `NeverVane/nanobrew` (Zig 0.15+)  
> **Binary:** `nb` — single static binary, ~1.2 MB  
> **Platforms:** macOS (arm64/x86_64), Linux (aarch64/x86_64 musl)  
> **Version documented:** 0.1.083

---

## Table of Contents

1. [High-Level Summary](#1-high-level-summary)
2. [Repository Layout](#2-repository-layout)
3. [Module-by-Module Primer](#3-module-by-module-primer)
4. [Install Pipeline (Homebrew Bottles)](#4-install-pipeline-homebrew-bottles)
5. [Install Pipeline (.deb Packages)](#5-install-pipeline-deb-packages)
6. [Dependency Graph](#6-dependency-graph)
7. [Runtime Directory Layout](#7-runtime-directory-layout)
8. [Concurrency Model](#8-concurrency-model)
9. [Platform Abstractions](#9-platform-abstractions)
10. [Adoption Checklist](#10-adoption-checklist)

---

## 1. High-Level Summary

nanobrew is a **drop-in Homebrew replacement** written entirely in Zig. It installs pre-built Homebrew bottles, macOS casks, third-party tap formulae, and Debian `.deb` packages — all using a single static binary with no Ruby runtime.

**Core design goals and techniques:**

| Goal | Technique |
|------|-----------|
| Fast warm installs | Content-addressable store (SHA256-keyed); APFS `clonefile(2)` / btrfs reflink |
| Parallel everything | OS threads per package; up to 16 concurrent in a sliding window |
| No subprocess sprawl | Native Zig HTTP client; native Mach-O / ELF header parsing |
| Safe state | Streaming SHA256 verification; atomic tmp→rename; JSON-escaped database writes |
| Drop-in compatibility | Same bottle URLs, same Cellar layout (`/opt/nanobrew/prefix/Cellar`) |
| Linux / Docker | Full native `.deb` pipeline: APT index fetch → dep resolve → ar extract → postinst |

---

## 2. Repository Layout

```
nanobrew/
├── build.zig                  # Zig build script — exe, lib module, cross-compile targets
├── src/
│   ├── main.zig               # CLI entry point — command dispatch
│   ├── root.zig               # Library root — re-exports all modules
│   ├── version.zig            # Version string comparison
│   ├── security_test.zig      # Adversarial security test suite
│   │
│   ├── api/                   # Homebrew API layer
│   │   ├── client.zig         # Formula + Cask JSON API client (cache-aware)
│   │   ├── formula.zig        # Formula struct + bottle tag logic
│   │   ├── cask.zig           # Cask struct + artifact types
│   │   ├── search.zig         # Search API (formulae + casks)
│   │   └── tap.zig            # Third-party tap Ruby formula/cask parser
│   │
│   ├── resolve/
│   │   └── deps.zig           # BFS dependency resolver + Kahn topological sort
│   │
│   ├── net/
│   │   ├── downloader.zig     # Parallel bottle downloader (GHCR token, SHA256 stream)
│   │   └── fetch.zig          # Low-level HTTP GET helper
│   │
│   ├── store/
│   │   ├── store.zig          # Content-addressable store (SHA256 → extracted keg)
│   │   └── blob_cache.zig     # Blob cache (downloaded .tar.gz bottles)
│   │
│   ├── cellar/
│   │   └── cellar.zig         # COW materialize from store into Cellar; keg detection
│   │
│   ├── linker/
│   │   └── linker.zig         # Symlink keg/bin/* → prefix/bin/, keg → prefix/opt/<name>
│   │
│   ├── db/
│   │   └── database.zig       # JSON state DB (kegs, casks, debs, history, pins)
│   │
│   ├── extract/
│   │   ├── tar.zig            # Bottle tar.gz extraction (shells to system tar)
│   │   └── native_tar.zig     # Pure-Zig POSIX tar parser (used by deb extractor)
│   │
│   ├── build/
│   │   ├── source.zig         # Source-build pipeline (cmake/autotools/meson/make)
│   │   └── postinstall.zig    # Post-install script runner
│   │
│   ├── cask/
│   │   └── install.zig        # macOS cask installer (.dmg/.zip/.pkg)
│   │
│   ├── platform/              # Platform abstraction layer
│   │   ├── platform.zig       # Feature flags (is_linux, is_macos, deb_arch)
│   │   ├── paths.zig          # All path constants (ROOT, CELLAR_DIR, etc.)
│   │   ├── copy.zig           # COW copy: clonefile(2) on macOS, reflink on Linux
│   │   ├── relocate.zig       # Dispatcher → macho/relocate or elf/relocate
│   │   └── placeholder.zig    # @@HOMEBREW_*@@ text-file substitution
│   │
│   ├── macho/
│   │   └── relocate.zig       # Mach-O header parser + install_name_tool + batch codesign
│   │
│   ├── elf/
│   │   └── relocate.zig       # ELF header parser + patchelf + text config substitution
│   │
│   ├── deb/                   # .deb package pipeline (Linux)
│   │   ├── index.zig          # APT Packages index parser (RFC 822, ArenaAllocator)
│   │   ├── resolver.zig       # Deb dependency resolver (alternatives, virtual packages)
│   │   ├── extract.zig        # .deb ar-archive extractor → native_tar
│   │   └── distro.zig         # Distro + arch auto-detection (/etc/os-release)
│   │
│   ├── services/
│   │   ├── services.zig       # Dispatcher → launchd or systemd
│   │   ├── launchd.zig        # macOS launchd plist discovery + start/stop
│   │   └── systemd.zig        # Linux systemd .service discovery + start/stop
│   │
│   ├── kernel/
│   │   ├── simd_scanner.zig   # Comptime SIMD byte scanner (ported from zigrep)
│   │   └── mmap_reader.zig    # mmap-backed zero-copy file reader
│   │
│   ├── mem/
│   │   └── arena.zig          # Arena allocator wrapper
│   │
│   └── exec/
│       ├── thread_pool.zig    # Thread pool (MPMC work queue)
│       └── dir_queue.zig      # Directory walker queue
│
├── worker/                    # Cloudflare Worker (nanobrew.trilok.ai)
│   ├── src/index.js           # Install script server + landing page
│   └── wrangler.toml          # Cloudflare deployment config
│
├── bench/                     # Benchmarking suite
│   ├── bench.sh               # Benchmark script
│   └── Dockerfile             # Docker benchmark environment
│
├── tests/
│   ├── smoke-test.sh          # End-to-end smoke tests
│   └── deb-parity.sh          # Deb vs apt-get parity tests
│
├── Formula/nanobrew.rb        # Homebrew tap formula (for `brew install nanobrew`)
├── install.sh                 # Build-from-source install script
├── .github/workflows/
│   ├── ci.yml                 # CI: build + test on macOS + Linux
│   ├── benchmark.yml          # Weekly benchmark runs
│   └── release.yml            # Release: cross-compile all targets + upload
├── BENCHMARKS.md
├── CHANGELOG.md
└── SECURITY.md
```

---

## 3. Module-by-Module Primer

### `src/main.zig` — CLI Entry Point

The top-level command dispatcher. Defines a `Command` enum (24 commands) and `Phase` enum (for per-package progress tracking). Key responsibilities:

- Parses `argv[1]` via an inline static dispatch table (aliases: `i`=install, `ui`=remove, `s`=search, `dr`=doctor, etc.)
- Validates package names via `isPackageNameSafe()` before any network or filesystem operation
- Runs `checkForUpdate()` non-blockingly after every command (once per day TTL)
- Drives the **sliding-window parallel install loop**: up to 16 threads run concurrently using ordered-removal of the oldest finished thread
- Renders a live TTY progress UI with braille spinners when stdout is a terminal; falls back to plain line-by-line output otherwise

### `src/root.zig` — Library Module Root

A pure re-export module. `build.zig` compiles this as the `nanobrew` module that `main.zig` imports. Also forces Zig's test runner to discover tests in all sub-modules via `comptime { _ = module; }` blocks.

### `src/api/client.zig` — Homebrew JSON API Client

Fetches formula and cask metadata from `formulae.brew.sh/api/`. Key behaviours:

- **Cache-first**: checks `~/.nanobrew/cache/api/<name>.json` with a 1-hour TTL before hitting the network
- **Tap detection**: if the name contains exactly 2 slashes (`user/tap/formula`), delegates to `tap.zig`
- **Shared client**: `fetchFormulaWithClient` accepts an optional `*std.http.Client` to reuse TLS connections across BFS levels
- **JSON parsing**: uses `std.json.parseFromSlice` with `std.json.Value` (schema-free); extracts bottle URL + SHA256 for the current platform tag (`BOTTLE_TAG`) with fallback array (`BOTTLE_FALLBACKS`)
- Respects `NANOBREW_API_DOMAIN` / `HOMEBREW_API_DOMAIN` env vars for mirror support

### `src/api/formula.zig` — Formula Struct

```
Formula {
  name, version, revision, rebuild, desc
  dependencies      []const u8   // runtime deps
  build_deps        []const u8   // build-time deps
  bottle_url        []const u8   // GHCR bottle URL
  bottle_sha256     []const u8   // expected SHA256 (64 hex chars)
  source_url        []const u8   // upstream source tarball URL
  source_sha256     []const u8
  caveats           []const u8
  post_install_defined  bool
}
```

`BOTTLE_TAG` is set at comptime from `builtin.os.tag` + `builtin.cpu.arch` (e.g. `arm64_sonoma`). `BOTTLE_FALLBACKS` lists older macOS tags and `all` as fall-backs.

### `src/api/cask.zig` — Cask Struct

```
Cask {
  token, name, version, url, sha256, homepage, desc
  auto_updates  bool
  min_macos     ?[]const u8
  artifacts     []Artifact    // .app, .binary, .pkg, .uninstall
}
```

### `src/api/search.zig` — Search API

Fetches the full formula and cask listing from `formulae.brew.sh/api/formula.json` (and cask variant), filters by substring match, and prints ranked results.

### `src/api/tap.zig` — Third-Party Tap Parser

Converts `user/tap/formula` → `github.com/user/homebrew-tap/Formula/formula.rb` (also tries `/Casks/`). Downloads the Ruby source file and parses it with a hand-rolled line scanner (no Ruby interpreter) that extracts: `version`, `url`, `sha256`, `depends_on`, `bottle do` blocks, `app`/`binary`/`pkg` artifact stanzas. Handles platform guards (`on_macos`, `on_linux`) using a depth counter.

### `src/resolve/deps.zig` — BFS Dependency Resolver

`DepResolver` struct holds two hash maps (`formulae: StringHashMap(Formula)` and `edges: StringHashMap([][]const u8)`) plus a shared `std.http.Client`.

- **`resolve(name)`**: BFS — seeds a frontier queue, then per level spawns one thread per unknown formula to call `api.fetchFormula()`, joins all threads, queues newly discovered deps
- **`topologicalSort()`**: Kahn's algorithm. Pre-flight check catches missing deps before in-degree computation to give `MissingDependency` vs `DependencyCycle`. Self-edges (packages that list themselves as deps, e.g. `r`, `neomutt`) are skipped
- Closes the shared HTTP client before returning sorted order (frees TLS resources)

### `src/net/downloader.zig` — Parallel Bottle Downloader

Three public types:

1. **`DownloadRequest`** — URL + expected SHA256
2. **`ParallelDownloader`** — queues up to 8 concurrent downloads via atomic work-stealing index
3. **`StreamingInstaller`** — download + extract in one shot per package (one thread per package)

`downloadOne()`:
- Rewrites GHCR bottle URLs if `NANOBREW_BOTTLE_DOMAIN` / `HOMEBREW_BOTTLE_DOMAIN` is set
- Fetches a GHCR bearer token from `ghcr.io/token?scope=repository:<repo>:pull` and caches it for 4 minutes in `cache/tokens/`
- Opens native `std.http.Client` per thread (thread-local, no sharing)
- Streams body through a `HashedReader` that computes SHA256 inline — no second pass
- Writes to `cache/tmp/<sha256>.dl` then atomic `rename` to `cache/blobs/<sha256>`

### `src/net/fetch.zig` — Low-Level HTTP GET

Thin wrapper around `std.http.Client` for simple GET requests. Used by API client and tap fetcher.

### `src/store/store.zig` — Content-Addressable Store

Maps SHA256 → extracted keg directory at `/opt/nanobrew/store/<sha256>/`.

| Function | Description |
|----------|-------------|
| `ensureEntry(alloc, blob_path, sha256)` | Extracts blob if not yet in store |
| `hasEntry(sha256)` | O(1) filesystem access check |
| `entryPath(sha256, buf)` | Formats the store path |
| `removeEntry(sha256)` | Deletes the store directory |

### `src/cellar/cellar.zig` — Cellar Materialization

Copies (COW where possible) a keg from the store into `/opt/nanobrew/prefix/Cellar/<name>/<version>/`.

- `materialize(sha256, name, version)`: uses `copy.cloneTree()` (macOS `clonefile`) or falls back to `copy.cpFallback()` (`cp --reflink=auto` on Linux)
- `detectKegVersion()`: walks the Cellar parent dir to find the actual installed version (handles `_rebuild` suffixes)
- `remove(name, version)`: deletes the keg dir; removes parent if empty

### `src/linker/linker.zig` — Binary Symlinker

- `linkKeg(name, version)`: walks `keg/bin/` and `keg/sbin/`, symlinks each file into `prefix/bin/`; creates `prefix/opt/<name>` → keg dir symlink
- `unlinkKeg(name, version)`: verifies each symlink still points to this keg before deleting (prevents removing another package's binary)

### `src/db/database.zig` — JSON State Database

File: `/opt/nanobrew/db/state.json`

Schema:
```json
{
  "kegs": [{"name":"...", "version":"...", "sha256":"...", "pinned":false, "installed_at":0}],
  "casks": [{"token":"...", "version":"...", "apps":[], "binaries":[]}],
  "history": {"<name>": [{"version":"...", "sha256":"...", "installed_at":0}]},
  "deb_packages": [{"name":"...", "version":"...", "sha256":"...", "installed_at":0, "files":[]}]
}
```

All writes use a hand-rolled JSON serializer (`writeJsonEscaped`) that escapes `"`, `\`, control characters, and null bytes — prevents JSON injection via malicious package names. The file is `sync()`-d and atomically replaced on every write.

### `src/extract/tar.zig` — Bottle Tar Extractor

Shells out to system `tar xzf <blob> -C <store_dir>`. Marked as v0 (correctness-first); a v1 using mmap + `std.compress.flate` is planned.

### `src/extract/native_tar.zig` — Pure-Zig POSIX Tar Parser

Used exclusively by the `.deb` pipeline. Parses POSIX ustar headers in memory without spawning any subprocess. Returns a list of extracted file paths (caller-owned).

### `src/build/source.zig` — Source Build Pipeline

Fallback when a formula has no bottle. Steps:
1. Download source tarball via `fetch.download()`
2. SHA256 verify via `shasum -a 256`
3. Auto-detect build system: probe for `CMakeLists.txt` → cmake, `configure` → autotools, `meson.build` → meson, `Makefile` → make
4. Run configure + make + make install into the Cellar path

### `src/build/postinstall.zig` — Post-Install Hook Runner

Handles platform-specific post-install steps (e.g. `ldconfig` for shared library registration). Limited subset of Homebrew's Ruby `post_install` — most common cases only.

### `src/cask/install.zig` — macOS Cask Installer

Handles three artifact types:
- **`.dmg`**: mounts with `hdiutil attach -nobrowse -quiet`, copies `.app` to `/Applications`, unmounts
- **`.zip`**: unzips to temp, copies `.app`
- **`.pkg`**: runs `installer -pkg <file> -target /`

Strips `com.apple.quarantine` extended attribute so apps open without Gatekeeper prompts. Creates `prefix/Caskroom/<token>/<version>/` metadata directory. Records in database.

### `src/platform/paths.zig` — Path Constants

All path constants are `comptime` string literals:

| Constant | Value |
|----------|-------|
| `ROOT` | `/opt/nanobrew` |
| `PREFIX` | `/opt/nanobrew/prefix` |
| `CELLAR_DIR` | `/opt/nanobrew/prefix/Cellar` |
| `BIN_DIR` | `/opt/nanobrew/prefix/bin` |
| `OPT_DIR` | `/opt/nanobrew/prefix/opt` |
| `STORE_DIR` | `/opt/nanobrew/store` |
| `BLOBS_DIR` | `/opt/nanobrew/cache/blobs` |
| `TMP_DIR` | `/opt/nanobrew/cache/tmp` |
| `API_CACHE_DIR` | `/opt/nanobrew/cache/api` |
| `TOKEN_CACHE_DIR` | `/opt/nanobrew/cache/tokens` |
| `DB_PATH` | `/opt/nanobrew/db/state.json` |

Homebrew placeholders and their real replacements are also defined here (`@@HOMEBREW_PREFIX@@` → `/opt/nanobrew/prefix`, etc.).

### `src/platform/copy.zig` — COW Copy Abstraction

```
cloneTree(src, dst) bool    // macOS: clonefile(2) syscall; Linux: always false
cpFallback(src, dst)        // macOS: cp -R; Linux: cp --reflink=auto -R
```

`clonefile` is declared `extern "c"` and only compiled on macOS via `comptime builtin.os.tag != .macos` guard.

### `src/platform/relocate.zig` — Relocation Dispatcher

Comptime-selects between `macho/relocate.zig` and `elf/relocate.zig` based on OS tag.

### `src/platform/placeholder.zig` — Text Placeholder Substitution

Replaces `@@HOMEBREW_PREFIX@@`, `@@HOMEBREW_CELLAR@@`, `@@HOMEBREW_REPOSITORY@@`, `@@HOMEBREW_LIBRARY@@` in text files (shebangs, `.pc`, `.cmake`, `.la`). Handles read-only files (0o555 mode) by temporarily chmod-ing.

### `src/macho/relocate.zig` — Mach-O Relocator (macOS)

1. Walks `bin/`, `sbin/`, `lib/`, `libexec/`, `Frameworks/` in the keg
2. Reads first 4 bytes; validates `MH_MAGIC_64` / `MH_CIGAM_64` / `FAT_MAGIC` / `FAT_CIGAM`
3. For each load command (`LC_ID_DYLIB`, `LC_LOAD_DYLIB`, `LC_LOAD_WEAK_DYLIB`, `LC_REEXPORT_DYLIB`, `LC_RPATH`): checks if the path contains `@@HOMEBREW_PREFIX@@` or `@@HOMEBREW_CELLAR@@`
4. If found: calls `install_name_tool -change/-rpath` to rewrite; adds file to `modified` list
5. After all files: one `codesign --force -s -` call covering all modified binaries

**Savings**: 3N subprocess spawns (otool + install_name_tool + codesign per file) → N+1 spawns.

### `src/elf/relocate.zig` — ELF Relocator (Linux)

Mirrors Mach-O relocator:
1. Detects ELF magic (`0x7f 'E' 'L' 'F'`)
2. Uses `patchelf --set-rpath` for RPATH rewriting
3. Substitutes placeholders in `.pc`, `.cmake`, `.la`, `.sh`, `.cfg` text files
4. Checks for `patchelf` availability upfront; prints actionable error if missing

### `src/deb/index.zig` — APT Package Index Parser

Parses the RFC 822-style APT `Packages` text format (double-newline-separated stanzas). Uses a single `ArenaAllocator` — a one-shot `deinit()` frees all 70K+ parsed packages. Tracks `Package`, `Version`, `Depends`, `Provides`, `Filename`, `SHA256`, `Size`, `Description` fields per entry.

### `src/deb/resolver.zig` — Deb Dependency Resolver

Parses the `Depends:` field format (`pkg (>= ver), pkg2 | pkg3, ...`):
- Splits on `,` → dependency groups
- Splits each group on ` | ` → alternatives
- Picks first alternative present in the real package index or `provides_map` (virtual packages)
- Falls back to the first alternative if none found in index

`resolveTransitive()`: BFS starting from the requested packages, using the parsed index. Returns a topologically-sorted list.

### `src/deb/distro.zig` — Distro Auto-Detection

Reads `/etc/os-release` to detect:
- Distribution name (Ubuntu/Debian)
- Version codename (e.g. `noble`, `jammy`, `bookworm`)
- Architecture (from `builtin.cpu.arch` → `"amd64"` / `"arm64"`)

Builds APT mirror URLs for `main` + `universe` components.

### `src/deb/extract.zig` — .deb Extractor

A `.deb` is a POSIX `ar` archive. This module:
1. Parses `ar` magic + 60-byte headers natively
2. Identifies `control.tar.*` and `data.tar.*` members
3. Detects compression: zstd (native via `std.compress.zstd`) / gzip (native via `std.compress.flate`) / xz (subprocess fallback)
4. Decompresses into memory
5. Passes to `native_tar.extractToDir()` for actual file extraction

`runPostinst()`: extracts `control.tar` to a temp dir, finds and executes the `postinst` script.

### `src/services/` — Service Management

Comptime dispatch to `launchd.zig` (macOS) or `systemd.zig` (Linux). Discovers `.plist` / `.service` files in installed kegs, wraps `launchctl`/`systemctl` for start/stop.

### `src/version.zig` — Version Comparison

`compareVersions(a, b)` splits on `.` and `_` (Homebrew uses `_` for rebuild/revision suffixes), compares segments numerically. Handles exhausted segments as 0. Used by `nb outdated` and `nb upgrade`.

### `src/kernel/simd_scanner.zig` + `src/kernel/mmap_reader.zig`

Ported from the `zigrep` project. SIMD byte scanner uses Zig's `@Vector` comptime types for vectorized byte searches. `mmap_reader.zig` provides a zero-copy file view. Currently available for future use in the native tar/extraction pipeline.

### `worker/src/index.js` — Cloudflare Worker

Serves `nanobrew.trilok.ai`:
- `GET /install` → returns the bash install script (detects OS/arch, downloads binary from GitHub Releases, verifies SHA256)
- `GET /` → returns HTML landing page with benchmark charts
- `GET /apt-get` → Linux-focused landing page
- Caches the latest GitHub release tag for 5 minutes

---

## 4. Install Pipeline (Homebrew Bottles)

```
nb install ffmpeg
│
├─ 1. INPUT VALIDATION
│     isPackageNameSafe() — rejects path traversal, control chars, null bytes
│
├─ 2. DEPENDENCY RESOLUTION  [src/resolve/deps.zig]
│     DepResolver.resolve("ffmpeg")
│       BFS level 0: fetch formula "ffmpeg" in parallel
│       BFS level 1: fetch all transitive deps in parallel (one thread each)
│       ...
│     topologicalSort() — Kahn's algorithm, cycle + missing-dep detection
│
├─ 3. SKIP INSTALLED  [src/cellar/cellar.zig]
│     For each formula: check if Cellar/<name>/<version>/ exists
│     If yes: run relocate + placeholder heal + relink (idempotent repair)
│     If no:  add to install_order
│
├─ 4. PARALLEL INSTALL LOOP  [src/main.zig]
│     Sliding window of 16 concurrent threads
│     Each thread runs fullInstallOne():
│
│     ┌─ BOTTLE PATH ─────────────────────────────────────────────────────┐
│     │  a. Download  [src/net/downloader.zig]                            │
│     │     - Fetch GHCR bearer token (4-min cache)                       │
│     │     - Stream HTTP body → SHA256 hash → tmp file → atomic rename    │
│     │     - Skip if blob already in cache/blobs/<sha256>                │
│     │                                                                   │
│     │  b. Extract   [src/store/store.zig + src/extract/tar.zig]        │
│     │     - extractToStore(blob_path, sha256)                           │
│     │     - System tar xzf → /opt/nanobrew/store/<sha256>/             │
│     │     - Skip if store entry already exists                          │
│     │                                                                   │
│     │  c. Materialize [src/cellar/cellar.zig]                          │
│     │     - clonefile(store/<sha>/<name>/<ver>, Cellar/<name>/<ver>)   │
│     │     - Fallback: cp --reflink=auto / cp -R                        │
│     └───────────────────────────────────────────────────────────────────┘
│
│     ┌─ SOURCE BUILD PATH ───────────────────────────────────────────────┐
│     │  (only when bottle_url is empty, source_url is present)          │
│     │  a. Download source tarball                                       │
│     │  b. SHA256 verify                                                 │
│     │  c. Detect build system (cmake/autotools/meson/make)             │
│     │  d. configure + make + make install → Cellar/<name>/<version>/   │
│     └───────────────────────────────────────────────────────────────────┘
│
│     After bottle or source:
│     d. Relocate  [src/platform/relocate.zig → macho/ or elf/]
│        macOS: parse Mach-O load commands → install_name_tool → batch codesign
│        Linux: parse ELF headers → patchelf → text-file substitution
│
│     e. Placeholder replace  [src/platform/placeholder.zig]
│        Replace @@HOMEBREW_*@@ in text files (shebangs, .pc, .cmake)
│
│     f. Link  [src/linker/linker.zig]
│        keg/bin/* → prefix/bin/  (symlinks)
│        keg → prefix/opt/<name>  (symlink)
│
├─ 5. PROGRESS UI
│     TTY:    braille spinners with phase labels per package, re-drawn at 80ms
│     Non-TTY: ✓ / ✗ lines printed after join
│
└─ 6. DATABASE RECORD  [src/db/database.zig]
      recordInstall(name, version, sha256) — serial (single file lock)
      Old version pushed to history[] before replacement
```

---

## 5. Install Pipeline (.deb Packages)

```
nb install --deb curl wget git
│
├─ 1. DISTRO DETECTION  [src/deb/distro.zig]
│     Parse /etc/os-release → codename (e.g. noble) + arch (amd64/arm64)
│
├─ 2. INDEX FETCH  [src/deb/index.zig]
│     HTTP GET Ubuntu/Debian Packages.gz for main + universe components
│     Decompress gzip → parse RFC 822 stanzas via ArenaAllocator
│     Build StringHashMap<DebPackage> (name → package)
│     Build provides_map: virtual package name → real package name
│     (NBIX binary cache: 70K packages in 32ms vs 3s for HTTP + decompress + parse)
│
├─ 3. DEPENDENCY RESOLUTION  [src/deb/resolver.zig]
│     parseDependsField() per package: handles "pkg | alt", "(>= ver)" constraints
│     resolveTransitive(): BFS, picks alternatives via index + provides_map
│     Topological sort → install order
│
├─ 4. PARALLEL DOWNLOAD + EXTRACT (8 threads)
│     For each deb in install_order:
│       a. Download .deb from mirror (streaming SHA256)
│       b. Extract .deb  [src/deb/extract.zig]
│          - Parse ar archive natively
│          - Decompress data.tar.{zst,gz,xz} natively (xz: subprocess)
│          - native_tar.extractToDir() → /
│          - Collect installed file paths
│       c. runPostinst() — control.tar → postinst script (if !--skip-postinst)
│          Common scripts: ca-certificates update-ca-certificates, ldconfig
│
└─ 5. DATABASE RECORD  [src/db/database.zig]
      recordDebInstall(name, version, sha256, files[]) per package
```

---

## 6. Dependency Graph

The following shows inter-module import relationships (→ means "imports"):

```
main.zig
  → root.zig (as "nanobrew" module)
  → std

root.zig (re-exports all):
  → api/client.zig
  → api/formula.zig
  → api/cask.zig
  → api/search.zig
  → api/tap.zig
  → resolve/deps.zig
  → net/downloader.zig
  → net/fetch.zig
  → extract/tar.zig
  → extract/native_tar.zig
  → store/store.zig
  → store/blob_cache.zig
  → cellar/cellar.zig
  → linker/linker.zig
  → db/database.zig
  → build/source.zig
  → build/postinstall.zig
  → cask/install.zig
  → platform/platform.zig
  → platform/relocate.zig
  → deb/index.zig
  → deb/resolver.zig
  → deb/extract.zig
  → deb/distro.zig
  → kernel/simd_scanner.zig
  → kernel/mmap_reader.zig
  → mem/arena.zig
  → exec/thread_pool.zig
  → services/services.zig
  → version.zig
  → security_test.zig

api/client.zig
  → api/formula.zig
  → api/cask.zig
  → api/tap.zig
  → net/fetch.zig
  → platform/paths.zig

api/tap.zig
  → api/formula.zig
  → api/cask.zig
  → net/fetch.zig

resolve/deps.zig
  → api/client.zig
  → api/formula.zig

net/downloader.zig
  → store/store.zig
  → platform/paths.zig

store/store.zig
  → extract/tar.zig
  → platform/paths.zig

cellar/cellar.zig
  → platform/paths.zig
  → platform/copy.zig

linker/linker.zig
  → platform/paths.zig

db/database.zig
  → platform/paths.zig

platform/relocate.zig    (dispatcher)
  → macho/relocate.zig  (macOS)
  → elf/relocate.zig    (Linux)

macho/relocate.zig
  → platform/paths.zig
  → platform/placeholder.zig

elf/relocate.zig
  → platform/placeholder.zig
  → platform/paths.zig

deb/extract.zig
  → extract/native_tar.zig
  → platform/paths.zig

deb/resolver.zig
  → deb/index.zig

services/services.zig   (dispatcher)
  → services/launchd.zig (macOS)
  → services/systemd.zig (Linux)

build/source.zig
  → api/formula.zig
  → net/fetch.zig
  → platform/paths.zig
```

**Layering summary** (no circular dependencies):

```
Layer 0 (leaf):   platform/paths.zig, version.zig, mem/arena.zig
Layer 1 (utils):  platform/copy.zig, platform/placeholder.zig,
                  kernel/*, exec/*, net/fetch.zig, extract/native_tar.zig
Layer 2 (data):   api/formula.zig, api/cask.zig, deb/index.zig
Layer 3 (I/O):    net/downloader.zig, extract/tar.zig, store/store.zig,
                  db/database.zig, deb/extract.zig, deb/distro.zig
Layer 4 (logic):  api/client.zig, api/tap.zig, api/search.zig,
                  resolve/deps.zig, deb/resolver.zig,
                  cellar/cellar.zig, linker/linker.zig,
                  macho/relocate.zig, elf/relocate.zig,
                  platform/relocate.zig
Layer 5 (ops):    build/source.zig, cask/install.zig, services/
Layer 6 (root):   root.zig (re-exports all)
Layer 7 (CLI):    main.zig
```

---

## 7. Runtime Directory Layout

```
/opt/nanobrew/
│
├── cache/
│   ├── blobs/          # Downloaded bottle .tar.gz files, keyed by SHA256
│   │   └── <sha256>    # Binary blob (never modified after atomic write)
│   ├── api/            # Cached formula/cask JSON responses (1-hour TTL)
│   │   ├── <name>.json
│   │   └── cask-<token>.json
│   ├── tokens/         # Cached GHCR bearer tokens (4-minute TTL)
│   │   └── homebrew_core_<pkg>
│   ├── apt/            # Cached APT package indices (deb mode)
│   └── tmp/            # Partial downloads (.dl extension → renamed on completion)
│
├── store/
│   └── <sha256>/       # Extracted keg tree (content-addressable, immutable)
│       └── <name>/
│           └── <version>/
│               ├── bin/
│               ├── lib/
│               └── ...
│
├── prefix/
│   ├── Cellar/         # Installed packages (COW copy from store)
│   │   └── <name>/
│   │       └── <version>/
│   ├── Caskroom/       # Installed casks
│   │   └── <token>/
│   │       └── <version>/
│   ├── bin/            # Symlinks → Cellar/<name>/<version>/bin/*
│   └── opt/            # Symlinks → Cellar/<name>/<version> (keg dir)
│
└── db/
    └── state.json      # JSON installation database
```

---

## 8. Concurrency Model

nanobrew uses plain OS threads (`std.Thread`) — no async runtime, no event loop.

### Install phase (bottle path)

```
main goroutine
│
├── Thread 1: fullInstallOne(pkg_0) ──→ download → extract → materialize → relocate → link
├── Thread 2: fullInstallOne(pkg_1) ──→ ...
├── ...up to 16 threads (sliding window)
└── Thread N: fullInstallOne(pkg_N-1)

Synchronization:
  - had_error: std.atomic.Value(bool)   — any thread sets on failure
  - phases[i]: std.atomic.Value(u8)     — phase enum for progress display
  - Database writes: serial (called after all threads join)
```

### Download phase (ParallelDownloader)

```
8 threads, all reading from:
  next_index: std.atomic.Value(usize)  — lock-free work-stealing
```

### Dependency resolution

```
Per BFS level:
  1 thread per unknown dependency name
  Each thread owns its own std.http.Client
  Results written into pre-allocated slots[] — no locks needed (disjoint indices)
  Main thread joins all before proceeding to next BFS level
```

### Progress rendering

The main thread calls `renderProgress()` which polls `phases[]` atomically at 80ms intervals, re-drawing N lines in-place using ANSI escape codes. The install threads run concurrently while the render loop spins.

---

## 9. Platform Abstractions

nanobrew targets macOS and Linux with zero conditional compilation in business logic — all platform differences are isolated behind comptime switches.

| Feature | macOS | Linux |
|---------|-------|-------|
| COW copy | `clonefile(2)` syscall (APFS) | `cp --reflink=auto` (btrfs/xfs) |
| Binary relocation | `macho/relocate.zig` + `install_name_tool` + `codesign` | `elf/relocate.zig` + `patchelf` |
| Service management | `launchd.zig` (`.plist` / `launchctl`) | `systemd.zig` (`.service` / `systemctl`) |
| Deb architecture | `"amd64"` or `"arm64"` (comptime) | same |
| Bottle tag | `arm64_sonoma` / `x86_64_monterey` / ... | `x86_64_linux` / `aarch64_linux` |

The `platform/platform.zig` module exposes:
```zig
pub const is_linux = builtin.os.tag == .linux;
pub const is_macos = builtin.os.tag == .macos;
pub const deb_arch = if (builtin.cpu.arch == .aarch64) "arm64" else "amd64";
```

Dead code for the non-target platform is eliminated at compile time — the macOS binary contains no ELF parsing, the Linux binary no Mach-O parsing.

---

## 10. Adoption Checklist

Use this checklist when embedding nanobrew in a new environment or contributing to the codebase.

### Initial Setup

- [ ] Install Zig 0.15 or newer (`zig version` to confirm)
- [ ] Clone the repo: `git clone https://github.com/justrach/nanobrew`
- [ ] Build: `zig build -Doptimize=ReleaseFast`
- [ ] Initialise directory tree: `sudo nb init` (chowns `/opt/nanobrew` to real user)
- [ ] Add to PATH: `export PATH="/opt/nanobrew/prefix/bin:$PATH"`
- [ ] Smoke test: `nb install tree && tree --version`

### macOS Prerequisites

- [ ] Xcode Command Line Tools (`xcode-select --install`) — provides `install_name_tool`, `codesign`, `tar`
- [ ] No additional runtime needed — single static binary

### Linux / Docker Prerequisites

- [ ] `patchelf` installed (`apt-get install patchelf`) — required for ELF relocation of Homebrew bottles
- [ ] `tar` in PATH — used by the bottle extractor (v0)
- [ ] For `.deb` mode: no extra deps — native ar/zstd/gzip parsing; only xz-compressed debs need `xz-utils`
- [ ] In Dockerfiles: use `nb install --skip-postinst` if postinst scripts cause failures in minimal images

### Migrating from Homebrew

- [ ] Run `nb migrate` — scans `/opt/homebrew/Cellar` and `Caskroom`, imports records into nanobrew's DB
- [ ] After migration: `nb list`, `nb outdated`, `nb upgrade` will see all existing packages
- [ ] Packages stay in `/opt/homebrew` — no conflict with existing Homebrew installation
- [ ] To fully switch: update your `PATH` to put `/opt/nanobrew/prefix/bin` before `/opt/homebrew/bin`

### CI/CD Integration

- [ ] Download pre-built binary from GitHub Releases (no Zig toolchain needed in CI)
- [ ] Cache `/opt/nanobrew/cache/blobs/` and `/opt/nanobrew/store/` between runs for warm installs
- [ ] Set `NANOBREW_API_DOMAIN` to point at an internal mirror for air-gapped environments
- [ ] Set `NANOBREW_BOTTLE_DOMAIN` to redirect GHCR bottle downloads to an internal registry

### Brewfile / Bundle Usage

- [ ] `nb bundle dump` — exports `Nanobrew.lock` file (brew/cask lines)
- [ ] `nb bundle install` — installs from file; instant no-op when nothing has changed
- [ ] Supports basic `brew "pkg"` and `cask "pkg"` lines; does not support Ruby DSL or `mas` lines

### Testing

- [ ] `zig build test` — all unit tests (works natively on macOS; cross-compile for Linux via Docker)
- [ ] Per-module: `zig build test-api`, `test-deb-index`, `test-deb-resolver`, `test-version`, `test-search`, `test-security`, etc.
- [ ] End-to-end: `tests/smoke-test.sh` (requires a live nanobrew install)
- [ ] Deb parity: `tests/deb-parity.sh` (compares `nb --deb` output vs `apt-get` on Ubuntu)

### What nanobrew Does NOT Support

- [ ] Homebrew Ruby `post_install` blocks (most bottle packages don't need them)
- [ ] `brew tap` as a standalone command (taps are resolved inline from `user/tap/formula` syntax)
- [ ] Mac App Store (`mas`) integration
- [ ] Complex Ruby DSL in Brewfiles (conditional blocks, custom Ruby code)
- [ ] `HOMEBREW_INSTALL_FROM_API=0` source-only mode (bottles always preferred)

### Security Considerations

- [ ] All package names are validated by `isPackageNameSafe()` before any disk/network operation (rejects path traversal, control characters, null bytes)
- [ ] GHCR downloads are SHA256-verified inline during streaming — no second pass
- [ ] Database writes escape all string values to prevent JSON injection
- [ ] `SUDO_USER` for chown is validated against `[a-zA-Z0-9\-_.]` before use
- [ ] Self-update (`nb update`) downloads the new binary, verifies SHA256, then atomically replaces itself — not `curl | bash`
- [ ] Mach-O binary parsing validates magic bytes and load command sizes before rewriting (prevents binary corruption from malformed bottles)

---

*Generated from source at commit on branch `longhand/type-codebase-docs-model-sonnet-4-6-budg-f8476f1d`.*
