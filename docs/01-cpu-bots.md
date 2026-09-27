# Phase 1: CPU opponents

Goal: players can fight a competent CPU opponent offline, with difficulty
levels, including while they wait in the matchmaking queue (phase 3).

**Scope: Battle Network 6 only**: all four cartridges (US Gregar and Falzar,
JP Glaga and Falzer) and the BN6 mods in Tango's patch list. Other games come
later, if at all.

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
- statuses: invisibility, barriers, flinch or paralysis
- **form**, as two independent fields per player, because they combine:
  the active **Cross** (none or which one) and the **Beast** state (none,
  Beast Out with turns left, Beast Over). A Cross and Beast Out together
  make a **Cross Beast**, and a single "form" value couldn't represent
  that. Rules (BN6):
  - A Cross Beast takes **two chip selects**: Cross on one turn, then Beast
    Out on a later one, or Beast Out first and Cross later. Never both on
    the same chip-select screen.
  - Once a Cross Beast is active, **every later chip select may pick a
    Cross again** (switching crosses) until Beast Out runs out.
- the custom gauge level, and on the custom screen which Crosses and
  whether Beast Out are on offer. Read these from the game's own
  availability flags rather than re-deriving the rules above, so mods
  that change them stay correct.
- the opponent's visible actions: attack animation starting, chip in use

tango-ai's BN6 addresses (see [prior-work.md](prior-work.md)) are a starting
point for those, but they came from 2024 Tango and one cartridge. Check each
one against the current build before trusting it.

**Four cartridges, four address tables.** `tango-gamesupport-bn6/src/pvp.rs`
already keeps per-ROM offsets (`PVP_BR5E_00`, `PVP_BR6E_00`, `PVP_BR5J_00`,
`PVP_BR6J_00`, with an `EWRAMOffsets` table and a `RawUnit` struct). Add the
bot's new fields there, for all four, so the observation code is shared.

**Mods.** Tango plays a mod with its base cartridge's offsets, so most mods
keep the same memory layout. But mods change chips and mechanics:

- Read chip data from the **patched ROM** (the dataview's
  `load_rom_assets_fn` already reads from the ROM being played), never from a
  fixed table.
- Heavy mods (BA Crossover, All-Stars, Legend of NetBattles) add chips and
  mechanics. Rule and look-ahead bots adapt, because they read chip data and
  simulate the real game. A trained model only knows the versions it was
  trained on.
- Test the bot's observation on each popular mod; the matchmaking board shows
  which ones people actually play.

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

0. ✅ **Fork and build.** Fork `tangobattle/tango`, build it on Windows
   (see [windows-setup.md](windows-setup.md)), run BN6 training mode.
1. ✅ **Scripted opponent in training mode.** Replace `dummy = 0` with a
   pluggable `Opponent` trait called each tick with the latest observation.
   First implementation: move randomly and fire the buster. This proves the
   plumbing. Done on the fork's `cpu-bots` branch (2026-09-27); see
   [dev-workflow.md](dev-workflow.md) for what the headless probe found.
2. ✅ **Richer BN6 observation.** Extend the BN6 poller with the fields above,
   including Cross and Beast state and the custom gauge. Verify each
   address with the probe, and write a debug overlay that shows what the
   bot sees. Done 2026-09-27 (`Bn6Obs`, the `bn6_explore` probe, and the
   "CPU sees" overlay; confirmed in play on Gregar). The RAM map is in
   [dev-workflow.md](dev-workflow.md).
   - **2b. Emotions, NaviCust bugs, panels and statuses** (next):
     emotion state for both players (Full Synchro, Anger, Tired,
     Exhausted), the opponent's only as far as it shows on screen; the
     bot's own NaviCust bugs, read from its save's NaviCust layout with
     Tango's existing save parsing (never the opponent's, which is
     hidden); barriers, invisibility, flinch and paralysis.
     **The whole field**, all visible to both sides: every panel's type
     (normal, cracked, broken, empty, grass, volcano, poison and the rest)
     and owner (stolen areas); rail/conveyor panels and which way they
     move; and every stage object (RockCube, IceCube, bombs, fans and so
     on) with its tile and HP. Lead: tango-ai's panel table at
     `0x02039C06` (`0x20` per column, `0x100` per row, owner in the next
     byte); objects likely sit in the object list beside the units.
     Also close milestone 2's open checks: player vs unit slot for the
     charge and form tables (a best-of-3), Falzar's Cross and Beast Over
     values, and the JP cartridges.
3. **Rules bot.** Chip-select heuristics plus reactive dodging. Picks a
   Cross or Beast Out, and plans a Cross Beast over two turns (either
   order), then re-picks the Cross each turn while Beast Out lasts,
   choosing it by element against the opponent's current form. Plays around
   the opponent's form. Opens the custom screen when the gauge is full, or
   holds it on purpose. Add a difficulty setting.
4. **"Vs CPU" session kind.** Best-of-N, bot's own save, a menu entry in the
   desktop UI.
5. **Look-ahead at chip select** using state save/restore. Compares
   candidate hands, including Cross, Beast Out and Cross Beast orders, by
   playing each out in the real game, so form interactions (and mods that
   change them) need no hand-written rules. Form choices pay off over
   turns (a Cross Beast needs two chip selects), so their look-ahead must
   span at least two turns, not one.
6. **Replay dataset pipeline.** Re-simulate replays headless in parallel and
   dump per-frame observations and actions.
7. **Imitation model** trained on the 4090 in WSL2, run in-game through
   ONNX, compared against the rules bot.
8. **Self-play** if the imitation model is promising.
9. **All four cartridges and the popular mods.** Check the observation on
   each and fix per-ROM offsets.
10. **Queue integration**: fight a CPU while waiting for a public match
    (see phase 3).

## Open questions

- Will upstream accept a "vs CPU" mode? Ask on the N1GP Discord before
  milestone 4. Milestones 0 to 3 are worth doing either way.
- Where to get a large replay collection for training. The community shares
  replays on Discord; check whether there's an archive.
