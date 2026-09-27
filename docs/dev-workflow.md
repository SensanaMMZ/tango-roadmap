# Dev workflow on the Windows machine

How the Tango fork is built, tested and probed day to day. Set up once
with [windows-setup.md](windows-setup.md); this is what happens after.
Everything here was run and verified on 2026-09-27.

## Layout

| What | Where |
| --- | --- |
| Tango fork (`SensanaMMZ/tango`) | `%USERPROFILE%\tango` on the Windows filesystem. `origin` is the fork, `upstream` is `tangobattle/tango` |
| This planning repo | WSL, `~/tango/tango-roadmap` |
| Build wrapper | `%USERPROFILE%\tango-env.cmd` (contents below) |
| Tango's data (saves, patches, logs) | `Documents\Tango` (`saves\`, `patches\`, `logs\tango.log`) |
| ROMs | Read straight out of the Legacy Collection install; nothing to copy |
| Build and test logs | `%USERPROFILE%\tango-build.log`, `tango-checks.log`, `tango-probe.log` |

Claude Code runs in WSL and drives the Windows toolchain through
`cmd.exe`. The source stays on the Windows side: MSVC builds from a WSL
path (`\\wsl$`) don't work, and cross-filesystem I/O is slow.

## The build wrapper

`tango-env.cmd` sets up the same environment as upstream's
`win/build.sh`, then runs its arguments in the fork's directory:

```bat
@echo off
rem Usage: tango-env.cmd <command...>   (runs in %USERPROFILE%\tango)
call "C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\VC\Auxiliary\Build\vcvars64.bat" >nul || exit /b 1
set "WG=%LOCALAPPDATA%\Microsoft\WinGet\Packages"
set "PATH=%USERPROFILE%\.cargo\bin;C:\Strawberry\perl\bin;C:\Program Files\LLVM\bin;C:\Program Files\CMake\bin;%WG%\Ninja-build.Ninja_Microsoft.Winget.Source_8wekyb3d8bbwe;%WG%\Google.Protobuf_Microsoft.Winget.Source_8wekyb3d8bbwe\bin;C:\Program Files\Git\cmd;%PATH%"
for /f "delims=" %%i in ('where cl.exe') do (set "PATH=%%~dpi;%PATH%" & goto :cl_done)
:cl_done
set CMAKE_POLICY_VERSION_MINIMUM=3.5
cd /d "%USERPROFILE%\tango" || exit /b 1
rem Below-normal priority: a full build saturates every thread and the
rem desktop (and games) stutter at normal priority.
start "" /belownormal /b /wait %*
exit /b %ERRORLEVEL%
```

Why each piece is there:

- **MSVC's `link.exe` first.** Other `link.exe`s on `PATH` (MSYS
  coreutils) shadow it otherwise.
- **Strawberry Perl.** `datachannel-sys` builds OpenSSL from source, and
  its `Configure` script needs Perl modules that MSYS Perl lacks. This is
  required, not optional.
- **Kitware CMake, LLVM, winget's Ninja and protoc ahead of devkitPro.**
  This machine has devkitPro's MSYS2 `git`, `cmake` and `perl` first on
  the global `PATH`. MSYS CMake breaks MSVC builds. The wrapper fixes
  the order for the build only, leaving the global `PATH` alone.
- **Below-normal priority**, plus `jobs = 24` in
  `%USERPROFILE%\.cargo\config.toml`, leaves 8 of 32 threads free. At full
  priority on every thread the desktop stuttered.
- **`exit /b %ERRORLEVEL%`**, so a failed build reports failure. Without
  it the wrapper returned 0 for a build that didn't finish.

## Commands

From WSL, always `cd` to a Windows path first: `cmd.exe` refuses a WSL
working directory. `$WINUSER` below is the Windows user name.

```bash
cd /mnt/c/Users/$WINUSER
# Release build of the desktop app (all games)
cmd.exe /c 'tango-env.cmd cargo build --locked --release --bin tango > %USERPROFILE%\tango-build.log 2>&1'
# Upstream's CONTRIBUTING checks, before any commit
cmd.exe /c 'tango-env.cmd cargo fmt --all -- --check'
cmd.exe /c 'tango-env.cmd cargo check --locked --bin tango --all-features'
cmd.exe /c 'tango-env.cmd cargo test --locked --lib -p tango-library --features tango-library/gamesupport-bn6 -p tango-session -p tango-gamesupport-common-ui'
cmd.exe /c 'tango-env.cmd python tools/check_workspace.py'
```

Use the Windows Git (`/mnt/c/Program Files/Git/cmd/git.exe`) in the fork,
so line endings and `core.longpaths` match the build.

Rules of thumb:

- **Run builds in the background** and read the log afterwards. Filter
  it to `error`, `warning`, `-->` and `Finished` lines rather than
  reading it whole.
- **Close Tango before a release build.** Windows won't replace a
  running `tango.exe`, so the build fails at the final step
  (`failed to remove file ...\tango.exe`). Ask the user to close it; don't
  kill it.
- **Stopping a build means stopping every `cargo.exe`.** There are two,
  the rustup proxy and the real Cargo. Killing only the proxy leaves the
  build running and holding the log file.
- **Don't edit `tango-env.cmd` during a build.** `cmd` re-reads batch
  files while running them, so an edit lands mid-command.
- **Don't pass paths with spaces through the wrapper.** The quoting
  breaks in `start ... %*`. Run built binaries directly instead, or copy
  the file to a path without spaces.

Baseline at upstream `84b2795`: all 54 tests in the command above pass.

## The headless probe

`tango-session/examples/custom_probe.rs` (branch `cpu-bots`) boots a real
BN6 training battle with no window. It reads the ROM from the Legacy
Collection and takes a save file as its argument, then drives both seats
from code and prints what each seat's custom screen did. Use it to answer
"what does the game do if…" questions instead of asking the user to
rebuild and click through a battle.

```bash
cmd.exe /c 'tango-env.cmd cargo build --release -p tango-session --example custom_probe'
cp "/mnt/c/Users/$WINUSER/Documents/Tango/saves/BN6 Gregar.sav" /mnt/c/Users/$WINUSER/AppData/Local/Temp/bn6g.sav
/mnt/c/Users/$WINUSER/tango/target/release/examples/custom_probe.exe 'C:\Users\<user>\AppData\Local\Temp\bn6g.sav'
```

It runs roughly 20 times faster than real time. Around a minute covers
eight 30-second battles.

## Findings so far

- **Upstream hides the Training button** (`35e39fae3`, 2026-07-24, a day
  after adding training). The session and its route were left intact; the
  fork restores the button in
  `tango-gamesupport-common-ui/src/editor/view/mod.rs`.
- **A BN6 link battle waits at the custom screen until both sides
  confirm.** Training's original dummy presses nothing, so training
  stalls at the first chip select. That's probably why the button was
  hidden.
- **Custom-screen controls (BN6, from the probe):** A alone, START alone,
  L, R or SELECT never close the screen. **A → START → A** closes it in
  about 25 ticks (pick a chip, START jumps the cursor to OK, A confirms).
  Walking the cursor right ×6 and pressing A also works (about 57 ticks).
- **The custom screen opens automatically once, at battle start.** After
  that it only opens when the player presses L or R with the gauge full.
  How long the gauge takes depends on the save (Custom1/Custom2 parts), so
  a bot must read it rather than assume a timer. Adding the gauge to the
  observation is part of milestone 2.
- **In a link battle, one player's L/R opens the custom screen for both
  players on the same tick.** Each side then confirms separately, and the
  battle resumes when both have.
- **The random bot gets through chip select on its own.** In a 90-second
  probe against a scripted player, it confirmed all 7 chip selects (in
  17 to 234 ticks) and dealt 255 damage. The custom screen reopened about
  every 720 ticks (12 s) with `BN6 Gregar.sav`, and that interval is the
  gauge fill time.

## Saving tokens

- Start a new Claude Code session (or `/clear`) between milestones. This
  file and `CLAUDE.md` carry the context forward.
- Batch in-game test feedback into one message.
- Prefer the probe over in-game testing for questions about game
  behaviour.
