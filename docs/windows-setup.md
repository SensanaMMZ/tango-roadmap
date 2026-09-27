# Windows setup

Two environments on the same machine:

- **Native Windows (MSVC)** for building and running Tango, matching upstream's
  release CI (`.github/workflows/release.yaml`, job `release-win32-x64`).
- **WSL2 (Ubuntu)** for the Python/ML side of bot training, with CUDA on the 4090.

## 1. Native toolchain for Tango

Install these (winget IDs shown where known; check with `winget search` if
one has changed):

| Tool | Why | Install |
| --- | --- | --- |
| Git | source, cargo git deps | `winget install Git.Git` |
| Visual Studio 2022 Build Tools, "Desktop development with C++" workload | MSVC compiler, linker, Windows SDK | `winget install Microsoft.VisualStudio.2022.BuildTools` then add the workload in the installer |
| LLVM (clang) at `C:\Program Files\LLVM` | melonDS's JIT won't compile with MSVC; `melonds-sys` looks for clang there, on `PATH`, or under `LLVM_ROOT` | `winget install LLVM.LLVM` |
| CMake | mGBA, melonDS, libdatachannel, mbedtls | `winget install Kitware.CMake` |
| Ninja | CMake generator for the melonDS core | `winget install Ninja-build.Ninja` |
| protoc | protobuf code generation | `winget install protobuf` (or download from the protobuf releases page and add to `PATH`) |
| Rust (stable, `x86_64-pc-windows-msvc`) | the project | `winget install Rustlang.Rustup`, then `rustup default stable-x86_64-pc-windows-msvc` |
| Strawberry Perl | **required**: `datachannel-sys` builds OpenSSL from source, and its scripts need Perl modules MSYS Perl lacks | `winget install StrawberryPerl.StrawberryPerl` |
| Python 3.11+ | upstream's workspace convention check (`tools/check_workspace.py`) | `winget install Python.Python.3.12` |
| GitHub CLI | forking, PRs | `winget install GitHub.cli` |
| Claude Code | continue this work | see <https://claude.com/claude-code> |

If another toolchain (devkitPro's MSYS2, Git Bash) puts its own `cmake`,
`perl` or `link.exe` ahead of these on `PATH`, the build picks the wrong
one. [dev-workflow.md](dev-workflow.md) has the wrapper script that fixes the
order for builds only.

Then:

```powershell
# Long paths: libdatachannel vendors files whose cargo checkout paths exceed 260 chars.
git config --global core.longpaths true
# Upstream CI sets this for older CMake projects in the dependency tree.
setx CMAKE_POLICY_VERSION_MINIMUM 3.5
```

Also enable Windows long paths system-wide (Group Policy "Enable Win32 long
paths", or the `LongPathsEnabled` registry value) if a checkout still fails.

## 2. Fork and build Tango

```powershell
gh repo fork tangobattle/tango --clone
cd tango
```

Build from a **"x64 Native Tools Command Prompt for VS 2022"** (or a
Developer PowerShell), so `cl.exe`, `link.exe` and the Windows SDK are on
`PATH`:

```powershell
cargo build --release --bin tango
```

The desktop crate's default feature is `gamesupport-all`, so every game is
included. Upstream's release build uses
`--features gamesupport-all --profile release-dist --target x86_64-pc-windows-msvc`.

If you build from Git Bash instead, MSYS's `link.exe` can shadow MSVC's.
Upstream's `win/build.sh` fixes this by putting `cl.exe`'s directory first
on `PATH`.

Useful checks (from upstream `CONTRIBUTING.md`):

```powershell
cargo fmt --all -- --check
cargo check --locked --bin tango --all-features
cargo test --locked --lib -p tango-library --features tango-library/gamesupport-bn6 -p tango-session -p tango-gamesupport-common-ui
```

You'll need BN6 from the Mega Man Battle Network Legacy Collection on Steam.
Tango finds the Legacy Collection install automatically.

## 3. WSL2 for training

```powershell
wsl --install -d Ubuntu
```

Inside Ubuntu:

- Install the NVIDIA **Windows** driver only. WSL2 shares it; don't install a
  Linux GPU driver inside WSL.
- Install Python and PyTorch with CUDA support (follow the current
  instructions at <https://pytorch.org/get-started/locally/>).
- Check with `nvidia-smi` and
  `python -c "import torch; print(torch.cuda.is_available())"`.

Keep training data on the WSL filesystem (`~/…`), not under `/mnt/c`.
Cross-filesystem I/O is much slower.

## 4. Resuming with Claude Code

```powershell
gh repo clone SensanaMMZ/tango-roadmap
cd tango-roadmap
claude
```

Claude Code loads `CLAUDE.md` automatically. Once the Tango fork exists,
work in the fork's directory and point Claude at this repo's docs.
