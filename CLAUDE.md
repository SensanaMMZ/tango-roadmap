# Context for Claude Code

This repo plans three additions to Tango (Mega Man Battle Network rollback
netplay, <https://github.com/tangobattle/tango>). The user wants them in this
order: **CPU bots → Android port (AYN Thor dual screen) → public matchmaking
with a hidden rating**.

**Scope for bots and matchmaking: BN6 only**, meaning all four cartridges and
the BN6 mods in Tango's patch list. When the user says "version" of BN6, they
mean a mod (or vanilla), not a cartridge. The user's decisions for
matchmaking:

- **US and JP share one pool.** The engine supports crossplay; the lobby's
  compatibility tag (keyed by family `bn6` vs `exe6`) is what separates them.
- **Anonymous device key is the default identity on every platform**
  (desktop, Tango Lite, Android). Discord linking is optional and later.
- **Wins and losses are reported automatically**, with no user input.
- **A live board** shows searching and playing counts per version, and players
  can fight a CPU while they wait.

Read `README.md` and the doc for the current phase before proposing work. `SESSION.md` records what was already asked and decided;
don't re-ask those questions.

## Where things stand

- No code has been written. The next step is phase 1, milestone 0: fork Tango
  and build it on the Windows machine (`docs/windows-setup.md`).
- The code will live in a fork of `tangobattle/tango` under the user's GitHub
  account (`SensanaMMZ`). This repo stays planning-only.
- The Windows machine (RTX 4090, 16C/32T, 64 GB) is the primary dev and training
  box. Use WSL2 for the Python/ML side; build Tango natively with MSVC as
  upstream CI does.

## Key facts to rely on (verified against upstream `84b2795`)

- **The CPU-bot hook already exists.** `tango-session/src/training.rs` runs a
  local two-core link battle with no network. At line ~266 the opponent's
  input is `let dummy = 0;`. A bot replaces that value each tick.
- **Per-game battle state is already read for every game except BCC.** `tango-match/src/telemetry.rs`
  defines `CorePoller`, `CoreObs` (both units' HP and tile, whether the
  custom screen is open) and `Event::ChipUsed`. Each game implements it in
  `tango-gamesupport-<game>/src/pvp.rs`. Extend this rather than writing new
  memory readers. The bot needs more fields (charge, hand, panels, statuses).
- **Rollback state save/restore is cheap**, which lets a bot look ahead by
  running the real game instead of a hand-written simulator.
- Tango has two frontends today: desktop (`tango/`, iced) and browser
  (`tango-lite-web/`, Dioxus + wasm). Everything below the frontend is
  portable. An Android app would be a third frontend modelled on
  `tango-lite-web`, not a port of the iced app.
- Multi-screen geometry lives in `tango-match/src/screens.rs` (`Arrangement`).
  Desktop stylus mapping is in `tango/src/session/stylus.rs` and needs to move
  to a shared crate for Android.
- Matchmaking today: a typed code becomes a session ID on
  `wss://matchmaking.tango.n1gp.net` (`tango-net/src/connect.rs`,
  `tango-lobby` `LinkIdent`). The server only pairs two sockets and relays
  WebRTC offers. A public queue just needs to hand both players a random
  session ID.

## Working preferences

- Match the upstream code style. Tango's code has dense doc comments explaining
  *why*; follow that.
- Keep bot, Android and matchmaking work in separate branches, so each can be
  offered upstream on its own.
- Don't let the bot read hidden information (the opponent's hand or folder
  order) even though game memory contains it.
