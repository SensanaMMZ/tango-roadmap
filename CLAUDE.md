# Context for Claude Code

This repo plans three additions to Tango (Mega Man Battle Network rollback
netplay, <https://github.com/tangobattle/tango>). The user wants them in this
order: **CPU bots → Android port (AYN Thor dual screen) → public matchmaking
with a hidden rating**.

**Scope for bots and matchmaking: BN6 only**, meaning all four cartridges and
the BN6 mods in Tango's patch list. When the user says "version" of BN6, they
mean a mod (or vanilla), not a cartridge. The user's decisions for
matchmaking:

- **Pools follow Tango's compatibility tags unchanged** (tag + match type).
  US (`bn6`) and JP (`exe6`) are different families, so they stay separate
  pools. Don't propose merging them.
- **Anonymous device key is the default identity on every platform**
  (desktop, Tango Lite, Android). Discord linking is optional and later.
- **Wins and losses are reported automatically**, with no user input.
- **A live board** shows searching and playing counts per version, and players
  can fight a CPU while they wait.

Read `README.md` and the doc for the current phase before proposing work.
`SESSION.md` records what was already asked and decided;
don't re-ask those questions.

## Where things stand

- **Phase 1, milestone 0 is done** (2026-09-27). The fork `SensanaMMZ/tango`
  is cloned on the Windows side, builds, and passes upstream's checks.
- **Milestones 1 and 2 are done** on the fork's `cpu-bots` branch: an
  `Opponent` trait in training (`tango-session/src/opponent.rs`), a random
  bot, the restored Training button, game-specific telemetry detail, BN6's
  `Bn6Obs` (`tango-gamesupport-bn6/src/observe.rs`), the `custom_probe` and
  `bn6_explore` examples, and a "CPU sees" debug overlay.
- **Next: milestone 2b** (emotions, NaviCust bugs, panels, statuses), then
  milestone 3, the rules bot. See `docs/01-cpu-bots.md`.
- **Read `docs/dev-workflow.md` before building anything.** It covers the
  build wrapper, the commands, the gotchas on this machine, the probe, and
  what's been learned about BN6's custom screen.
- This repo stays planning-only; code lives in the fork.
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
