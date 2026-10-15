# CLI distribution and runtime options (issue #4)

Question: for a CLI that manipulates files and symlinks and must be easy to install on macOS and Linux (ideally Windows), compare an npm package, a single static binary (Go / Rust / Bun-compiled), and shell/Python script distribution on install friction, symlink support, and maintenance cost.

Research date: 2026-10-15. Sources are official docs; claims are cited inline. Judgments are marked as such.

## Findings that apply to every option

- **Windows symlinks are the real cross-platform constraint, not the runtime.** Creating symlinks needs the `SeCreateSymbolicLinkPrivilege`, held by Administrators by default ([Microsoft Learn: Create symbolic links](https://learn.microsoft.com/en-us/windows/security/threat-protection/security-policy-settings/create-symbolic-links)). Python's docs add that unprivileged accounts can create symlinks on newer Windows 10 if Developer Mode is enabled ([CPython os.rst](https://github.com/python/cpython/blob/main/Doc/library/os.rst), `os.symlink`). Every language below hits this identically.
- **Windows distinguishes file and directory symlinks.** Python: "a symlink represents either a file or a directory, and does not morph to the target dynamically" (same source). Rust splits the API into `symlink_file` / `symlink_dir` ([std::os::windows::fs](https://doc.rust-lang.org/std/os/windows/fs/index.html)); Go's `os.Symlink` notes that a link to a non-existent target becomes a file symlink and breaks if the target is later a directory ([pkg.go.dev/os#Symlink](https://pkg.go.dev/os#Symlink)).
- **Node adds a no-privilege fallback for directories.** `fs.symlink` accepts type `'junction'`; junctions require an absolute destination (Node normalizes it) and "can only point to directories" on NTFS ([Node fs docs](https://nodejs.org/api/fs.html#fssymlinktarget-path-type-callback)). With `type` null Node autodetects file/dir, defaulting to file if the target is missing (same page). So Node/Bun can degrade to junctions for directory links on Windows without admin; Go, Rust and Python must implement that themselves. (Inference from the docs: the Node page documents junction support; the others' cited pages do not mention one.)
- **Symlink API surface is available on all three platforms in every candidate:** Node `fs.symlink`, Python `os.symlink`, Go `os.Symlink`, Rust `std::os::unix::fs::symlink` plus the Windows pair. Rust's unix and windows functions live in different modules, so cross-platform code needs `cfg` gates ([std::os::unix::fs::symlink](https://doc.rust-lang.org/std/os/unix/fs/fn.symlink.html)).

## Option 1: npm package

- **Install:** `npm i -g` / `npx`. Requires Node already installed; `engines` is advisory only unless `engine-strict` is set ([npm package.json docs](https://docs.npmjs.com/cli/v10/configuring-npm/package-json)).
- **Executables:** on Unix npm symlinks the `bin` file onto PATH; on Windows it generates a cmd shim (same source). The `bin` file needs a `#!/usr/bin/env node` shebang.
- **Symlinks:** `fs.symlink` with junction fallback (above) — best Windows story of the options.
- **Maintenance:** one codebase, one publish pipeline, no per-platform builds. Cost is the Node version matrix (users' Node versions vary) and npm supply-chain hygiene. The repo already has a `package.json`, so this matches the existing layout.
- **Friction:** zero for people who already have Node (likely for an AI-harness audience; this is a judgment); a prerequisite install for others.

## Option 2: single binary

### Bun `--compile`
- Bundles script + runtime into one executable; cross-compiles to `bun-linux-{x64,arm64}[-musl]`, `bun-windows-{x64,arm64}`, `bun-darwin-{x64,arm64}` ([Bun executables docs](https://bun.com/docs/bundler/executables)).
- Caveats from the docs: the binary is large (the page says it still needs to shrink), non-embedded files are read from disk at runtime, Windows metadata options cannot be used when cross-compiling, and codesigning guidance is macOS-only.
- Symlinks: Node-compatible `fs` API, so the junction approach carries over (Bun's Node-compat for `fs.symlink` was not separately verified against Bun docs).

### Node Single Executable Applications (SEA)
- Official Node feature but Stability 1.1 "Active development"; `--build-sea` added in v25.5.0. CI-tested on Windows, macOS arm64 only (x64 skipped), Linux except Alpine and s390x. Cross-building requires disabling code cache and snapshot; the Node binary used must match versions; module loading from disk is off by default ([Node SEA docs](https://nodejs.org/api/single-executable-applications.html)).

### Go
- Static binaries via `GOOS`/`GOARCH` cross-compilation to darwin/linux/windows (amd64, arm64 among others); `CGO_ENABLED=0` avoids a C toolchain ([Go install-from-source](https://go.dev/doc/install/source)).
- Symlinks: `os.Symlink`; Windows file/dir caveat above; no built-in junction fallback.

### Rust
- Native binaries; symlink API split by OS (above). Cross-compiling needs targets/linkers per platform. (Not verified against rustup docs; in practice CI builds on native runners instead.)

### Common binary costs
- Needs a release pipeline producing one artifact per OS/arch. GitHub-hosted runners exist natively for linux x64/arm64, windows x64/arm64, macOS arm64 and Intel ([GitHub Docs: choose the runner](https://docs.github.com/en/actions/how-tos/write-workflows/choose-where-workflows-run/choose-the-runner-for-a-job)), so a native matrix build is feasible without cross-compilation.
- Users need a way to fetch the right binary (curl installer, Homebrew tap, GitHub release download) — that is extra install surface to maintain. macOS needs signing (Node SEA docs show `codesign`; Bun docs cover macOS signing); Windows signing is optional for SEA.
- Friction for users: no runtime prerequisite, but the first-install step is a download rather than a package-manager one-liner.

## Option 3: shell / Python script

- **Shell:** runs everywhere on macOS/Linux with no runtime, but not on Windows without WSL/Git Bash (judgment; not source-backed). Symlink via `ln -s` is the most direct of all options on Unix. Portability and testability of non-trivial logic degrade quickly (judgment).
- **Python:** `os.symlink` with the Windows privilege/file-vs-dir notes above. Distribution as a package via `uv tool install` / `uvx` gives isolated envs and puts executables on PATH (symlinked on Unix, copied on Windows); requires uv and a Python it can find/manage, and the PATH dir may need `uv tool update-shell` ([uv tools docs](https://docs.astral.sh/uv/concepts/tools/)). Python is not guaranteed on every machine.

## Comparison

| | npm | Bun/Go/Rust binary | Shell | Python (uv/pipx) |
|---|---|---|---|---|
| Prerequisite | Node | none | none (Unix) | Python + uv |
| Windows | yes (cmd shim) | yes | poor | yes |
| Windows symlink no-admin dir fallback | junction built into `fs.symlink` | Bun: via Node API; Go/Rust: DIY | n/a | DIY |
| Release pipeline | `npm publish` | per-OS/arch matrix, signing, installer | copy a file | `pip`/PyPI publish |
| Maintenance | lowest | highest | low, but logic scales badly | low-medium |

## Recommendation (judgment)

1. **Ship as an npm package first** (`npx skillbag` / `npm i -g skillbag`). Lowest maintenance, repo already has a `package.json`, shim handling on Windows is done by npm, and Node's `fs.symlink` gives a junction fallback for the Windows no-admin directory case. Prerequisite (Node) is a reasonable assumption for this audience.
2. **Keep the code free of Node-only assumptions beyond `fs`/`path`** so it can later be `bun build --compile`d (or SEA'd) into standalone binaries attached to GitHub releases if no-Node users matter. This is an additive step, not a rewrite.
3. **Avoid Go/Rust unless** performance or a no-runtime story becomes a hard requirement; they double the symlink code paths for Windows and add a release matrix.
4. **Avoid pure shell** as the main implementation (no Windows, hard to test). A thin shell bootstrap that installs/launches the npm package is acceptable if desired.
5. **Design for Windows symlink failure regardless of runtime:** detect `EPERM`, fall back to junctions for directories, and for files either copy or print a Developer Mode hint.

## Gaps / not verified
- Bun's own docs on `fs.symlink` Windows behavior and binary size numbers were not found; Rust cross-compilation docs were not consulted.
- Shell-on-Windows and "audience has Node" statements are judgments, not sourced.
