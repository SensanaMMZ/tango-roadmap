# Phase 2: Android port with AYN Thor dual screen

Goal: Tango on Android with single-player, CPU bots, netplay compatible with
PC players, and replays. On the AYN Thor, DS games use both screens, and GBA
games use the bottom screen for match information.

No Android port of Tango exists, finished or started (checked 2026-09-27; see
[prior-work.md](prior-work.md)).

## Architecture: a third frontend, not a port of the desktop app

Tango has two frontends (desktop `tango/` on iced, browser `tango-lite-web/` on
Dioxus and wasm). Everything below them compiles for native and wasm. Don't
port the desktop app:

- iced on Android is immature and can't put a window on a second display.
- The desktop app pulls in desktop-only crates: `rfd` (file dialogs),
  `arboard` (clipboard), `discord-ipc`, `crash-handler`/`minidumper`,
  `directories-next`, Steam discovery.

Instead: a **Kotlin app** plus a **Rust `cdylib` crate (`tango-android`)** over
JNI. Model the Rust side on `tango-lite-web`, which already:

- drives sessions from an event loop (on Android: `Choreographer` and the
  audio callback),
- supplies storage and HTTP through `tango-library`'s `Storage` and `Http`
  traits (on Android: the app's files directory, and `reqwest` with rustls),
- owns its audio output, touch input and gamepad input.

## Cross-compiling the native code

Use `cargo-ndk` with target `aarch64-linux-android`.

| Dependency | How it builds | Android notes |
| --- | --- | --- |
| mGBA (`mgba-rs`/`mgba-sys`) | CMake | Should cross-compile with little work |
| melonDS (`melonds-rs`/`melonds-sys`) | CMake, partly invoked by hand from `build.rs` | Needs the NDK toolchain file and ABI passed explicitly. Turn the arm64 JIT on. Biggest build risk. |
| libdatachannel + mbedtls (`datachannel-facade`) | CMake, vendored | Should mostly work |
| `tango-library` `native` feature | uses `reqwest/native-tls` | Means OpenSSL on Android. Switch to rustls. |
| `gamepad-facade` | SDL3 by default | Disable default features. Thor controls arrive as Android gamepad events. |
| `cpal` | AAudio on Android | Works, or use Oboe |

## App responsibilities (Kotlin)

- ROM and save import through the Storage Access Framework. There's no Steam
  Legacy Collection on Android, so players copy files over from a PC.
- UI: game and save selection, lobby, settings (Jetpack Compose).
- Game surfaces, audio, input from `KeyEvent`/`MotionEvent`.

## AYN Thor dual screen

The Thor has a 6" 1080p 120 Hz AMOLED top screen and a 3.92" AMOLED bottom
touchscreen, on Android 13.

- The Thor treats the **top screen as the primary display** and the bottom as
  secondary. The AYANEO Pocket DS and Retroid Dual Screen are the other way
  round. Add a "swap screens" setting.
- Find the bottom screen with
  `DisplayManager.getDisplays(DISPLAY_CATEGORY_PRESENTATION)`, and show a
  `Presentation` on it holding its own `SurfaceView`. Reference
  implementation: the melonDS Android fork's `ExternalPresentation.kt` and
  `SecondaryDisplaySelector.kt`.
- `tango-match::screens::Arrangement` packs screens into one image. Add a
  per-screen mode that yields each screen's rectangle separately, so the DS
  top screen goes to the primary surface and the touch screen to the
  Presentation. Keep using `Arrangement` for touch placement so picture and
  touch area can't disagree.
- Touches on the bottom surface become the DS stylus. Move
  `tango/src/session/stylus.rs` into a shared crate so both hosts use it.
- Controller input goes to whichever window has focus, and the Presentation
  can take it. Forward input from both windows.
- **GBA games** have one screen. Use the bottom one for what the desktop shows
  beside the game: opponent view (an existing setting), netplay telemetry and
  frame delay, ready and lobby status, the results card, replay controls,
  input display.
- The top screen runs at 120 Hz. Pace emulation from audio and `Choreographer`,
  not from each 120 Hz frame.

## Compatibility with PC players

Use the same protocol version and matchmaking server as upstream desktop
builds, so Android players can match against PC players and replays work on
both. The lobby already refuses pairings whose `sim_version` differs, which
protects against a build that simulates differently.

## Milestones

1. The Rust core cross-compiles for Android and runs a GBA game headless in a
   test binary on the device.
2. One-screen Kotlin app: single-player and replay viewing.
3. Netplay against a PC.
4. CPU bots (from phase 1) on Android.
5. BN5 DS on two screens with stylus on the bottom.
6. GBA bottom-screen extras.

Test the two biggest risks first: the melonDS cross-compile, and whether DS
rollback runs fast enough on the Snapdragon 8 Gen 2.

## Quick check you can do today

Tango Lite (<https://lite.tango.n1gp.net/>) is the official browser build. It
has single-player, netplay, replays, touch controls and every game including
BN5 DS. Open it in Chrome on the Thor to gauge performance. It was not
verified to run on Android Chrome, and a browser tab only uses one screen.
