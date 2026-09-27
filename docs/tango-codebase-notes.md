# Tango codebase notes

Surveyed at upstream commit `84b2795` (2026-09-23). Upstream changes fast; re-check
paths before relying on them. Upstream's own `ARCHITECTURE.md` and
`tango-match/ARCHITECTURE.md` are the authoritative maps.

## Workspace

Rust workspace, GPL-3.0.

| Crate | Role |
| --- | --- |
| `tango` | Desktop frontend (iced 0.14, wgpu, cpal, SDL3 gamepads via `gamepad-facade`) |
| `tango-lite-web` | Browser frontend (Dioxus 0.7, wasm, WebAudio, IndexedDB). Not a default workspace member; build with its `build.sh` |
| `tango-library` | Game registry, catalogs, shared config, save operations, loadouts, stats. I/O through `Storage` and `Http` traits; `native` feature supplies std versions |
| `tango-lobby` | Connection negotiation, readiness, settings and save exchange |
| `tango-net`, `tango-net-protocol` | Transport (WebRTC data channels, direct UDP), wire types |
| `tango-platform` | Portable spawning and timers |
| `tango-session` | Session kinds (single-player, training, PvP, replay), drivers, audio streams |
| `tango-match` | Rollback coordination, backend contracts, telemetry, replay seeking, screen arrangement |
| `tango-backend-mgba`, `tango-backend-melonds` | GBA and DS emulation |
| `tango-gamesupport-<game>` (+ `-dataview`, `-ui`) | Per-game hooks, save layouts, editors. Games: bn1–bn6, bn5ds, exe45, exeoss, bcc |
| `tango-replay`, `tango-replay-renderer` | Replay format, video export |

External crates from the tangobattle org: `mgba-rs`, `mgba-rollback`,
`melonds-rs`, `melonds-rollback`, `getgud` (generic rollback core),
`datachannel-facade` (libdatachannel natively, browser WebRTC on wasm),
`gamepad-facade`, `encoder-facade`, `tango-signaling`, `tango-patch`, `rennet`.

## Files that matter for this roadmap

**Bots**

- `tango-session/src/training.rs`: local two-core battle; `let dummy = 0;`
  (~line 266) is the opponent input. One round only
  (`TRAINING_MATCH_TYPE = (0, 0)`); mirror match on the player's save.
- `tango-match/src/telemetry.rs`: `CorePoller`, `CoreObs`, `UnitObs` (HP,
  tile), `Event` (round start and end, match end, `ChipUsed`). Pollers are
  rollback-safe by cloning.
- `tango-gamesupport-bn6/src/pvp.rs`: BN6's poller and traps (about 900 lines).
- `tango-match/src/solo.rs`, `engine.rs`: backend and console contracts.

**Android**

- `tango-lite-web/src/`: `host.rs` (composition), `engine.rs` (pumping,
  screen composition), `audio.rs`, `input.rs` (touch, keyboard, gamepad),
  `storage.rs`, `http.rs`, `link.rs`. The template for a new host.
- `tango-match/src/screens.rs`: `Stacking`, `Arrangement`
  (`order`, `size`, `touch_screen_placement`, `rearrange`, `blit`).
- `tango/src/session/stylus.rs`: pointer to DS touch mapping (desktop only
  today).
- `tango-library/src/storage.rs`, `http.rs`: the traits a host implements.

**Matchmaking**

- `tango-library/src/config.rs`: `DEFAULT_MATCHMAKING_ENDPOINT =
  "wss://matchmaking.tango.n1gp.net"`.
- `tango-lobby`: `LinkIdent::{Matchmaking(String), Direct(DirectRole)}`,
  `State::reconcile`, handoff tickets.
- `tango-net/src/connect.rs`: `open_channels`, signaling rendezvous by
  `session_id`.
- `tango-net-protocol/src/control.rs`: `Settings`, `GameInfo`
  (includes `sim_version`), `NegotiatedState`.
- `tangobattle/tango-signaling`: client and `src/proto/signaling.proto`.

## Build facts

- Windows release: MSVC target, clang from `C:\Program Files\LLVM` for the
  melonDS core (its JIT isn't MSVC-compatible), Ninja, protoc,
  `CMAKE_POLICY_VERSION_MINIMUM=3.5`, `core.longpaths`. See
  [windows-setup.md](windows-setup.md).
- `cargo build --bin tango` defaults to all games (`gamesupport-all`).
- The portable core tests run without ROMs; see upstream `CONTRIBUTING.md`.
