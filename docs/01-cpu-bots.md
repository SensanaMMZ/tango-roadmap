# Phase 1: CPU opponents

Goal: players can fight a competent CPU opponent offline, with difficulty
levels. Start with **BN6** (the most-played competitive game, and the one
tango-ai already mapped), then extend game by game.

## Anatomy of a bot

A bot needs to **see** the game, **press buttons**, and **decide**. Tango
already provides the first two.

### Pressing buttons: training mode is the hook

`tango-session/src/training.rs` is "a netplay match with the network cut
out": two cores, both running the player's ROM and save, primed into a link
battle. The human drives one core. The other core's input is supplied locally
before each tick:

```rust
// tango-session/src/training.rs, ~line 265
// The dummy presses nothing.
let dummy = 0;
...
let core0 = if controlled == 0 { player } else { dummy };
let core1 = if controlled == 0 { dummy } else { player };
```

A "vs CPU" session is this session with `dummy` computed by a bot each tick.
There's no network, no rollback churn, and both cores run in lockstep. Things
to change beyond that line:

- Training fights one round (`TRAINING_MATCH_TYPE = (0, 0)`). A CPU match
  wants best-of-N, so it needs its own match type or its own session kind.
- Training mirrors the player's save on both cores. A CPU opponent should
  load its own save (a folder built for the bot and its difficulty).

### Seeing the game: extend the existing telemetry

`tango-match/src/telemetry.rs` already has a per-game `CorePoller` that reads
battle state every tick and is rollback-safe:

- `CoreObs`: both units' HP and tile, and whether this core's custom screen
  is open.
- Events: `RoundStarted`, `RoundEnded`, `MatchEnded`, `ChipUsed { player, chip }`.

Every game except BCC implements this in `tango-gamesupport-<game>/src/pvp.rs`.
The bot needs more:

- buster charge level, current chip in hand and its remaining uses
- the custom-screen hand (the chips offered, their codes, what's selected)
- panel states and ownership (cracked, broken, stolen areas)
- statuses: invisibility, barriers, flinch or paralysis, cross or beast form
- the opponent's visible actions: attack animation starting, chip in use

tango-ai's BN6 addresses (see [prior-work.md](prior-work.md)) are a starting
point for those, but they came from 2024 Tango. Check each one against the
current build before trusting it.

Per-game `-dataview` crates already parse folders and chip data (names,
damage, codes). Use them for chip knowledge instead of hand-copying tables.

### Deciding

From cheapest to strongest:

1. **Hand-written rules.** Pick chips by code compatibility, damage and
   Program Advances. Dodge when the opponent's attack starts. Charge the
   buster when nothing better is available. Use barriers and counters on
   reaction. Beats casual players, and costs a few weeks for one game.
2. **Rules plus look-ahead in the real game.** Rollback needs cheap
   save/restore of the entire game, so the bot can clone state, simulate a
   few seconds of each candidate, and keep the best. Use it for chip
   selection first (a few hundred frames per candidate hand), then short
   movement look-ahead in battle. tango-ai hand-wrote a BN6 simulator in
   Python to do search. Using the emulator gives exact game rules with no
   re-implementation of chips.
3. **Imitation learning from replays.** Tango replays are complete,
   deterministic input logs, so any recorded match can be re-simulated to
   recover full state at every frame. Train a small model on game-state
   features (not screenshots) to predict human actions.
4. **Self-play reinforcement learning**, then **learned model plus search**
   (the AlphaZero pattern) for a strong bot. Research-grade effort.

**Recommendation:** ship option 2 for BN6 first, then try option 3 to see
whether learning beats the rules.

## Fairness rules (non-negotiable)

- **No hidden information.** Game memory holds the opponent's hand and folder
  order. The bot's observation must be limited to what a human sees.
- **Human-like reaction.** Add reaction delay and cap inputs per second.
  Frame-perfect dodging feels unfair, not smart.
- **Difficulty levels** are the same bot with more delay, injected mistakes,
  weaker chip choices or less look-ahead.

## Where it runs

- **Training** happens on the Windows machine. The 4090 handles any model
  size a BN bot needs. The 16 cores matter more for self-play, because
  training speed is limited by emulator frames and mGBA runs on CPU. Plan on
  24 to 28 headless game instances in parallel.
- **Players** only run the finished bot. Rules and small models cost almost
  nothing. Look-ahead costs CPU, so on Android limit it to chip select.
  Ship trained models as files and run them from Rust with `tract` or `ort`
  (ONNX Runtime). No player GPU needed. Avoid screen-image models: they'd push
  a GPU requirement onto players.

## Milestones

0. **Fork and build.** Fork `tangobattle/tango`, build it on Windows
   (see [windows-setup.md](windows-setup.md)), run BN6 training mode.
1. **Scripted opponent in training mode.** Replace `dummy = 0` with a
   pluggable `Opponent` trait called each tick with the latest observation.
   First implementation: move randomly and fire the buster. This proves the
   plumbing.
2. **Richer BN6 observation.** Extend the BN6 poller with the fields above,
   and write a debug overlay that shows what the bot sees.
3. **Rules bot.** Chip-select heuristics plus reactive dodging. Add a
   difficulty setting.
4. **"Vs CPU" session kind.** Best-of-N, bot's own save, a menu entry in the
   desktop UI.
5. **Look-ahead at chip select** using state save/restore.
6. **Replay dataset pipeline.** Re-simulate replays headless in parallel and
   dump per-frame observations and actions.
7. **Imitation model** trained on the 4090 in WSL2, run in-game through
   ONNX, compared against the rules bot.
8. **Self-play** if the imitation model is promising.
9. **Second game** (BN5 or BN4), reusing the `Opponent` trait with a new
   observation mapping.

## Open questions

- Will upstream accept a "vs CPU" mode? Ask on the N1GP Discord before
  milestone 4. Milestones 0 to 3 are worth doing either way.
- Where to get a large replay collection for training. The community shares
  replays on Discord; check whether there's an archive.
