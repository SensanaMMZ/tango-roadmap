# Prior work

Searched 2026-09-27: GitHub (Tango issues, forks, repo search) and the web.

## No Android port exists

- Tango's repo has no Android build code. The only "android" mention is
  keyboard key-code parsing in `tango/src/platform/input.rs`.
- None of the roughly 20 forks of `tangobattle/tango` is an Android port.
  The active ones:
  - `HikariCalyx/trill`: a Chinese-server fork (Chinese locale, CN
    signaling server, Discord removed).
  - `rkoh46/tangoAW2`: rollback netplay for Advance Wars 2.
  - `indianaorz/tango-ai`: bot training (below).
- Web searches found nothing for Tango on Android.

## Tango Lite (official browser build)

<https://lite.tango.n1gp.net/>, source in `tango-lite-web/`. Single-player,
live netplay, patches, replays and video export, touch controls, every game
including BN5 DS (the DS backend needs cross-origin isolation for shared wasm
memory). One screen only. The closest thing to a mobile Tango today, and the
best template for an Android host.

## indianaorz/tango-ai (bot training experiment)

<https://github.com/indianaorz/tango-ai>. Description: "Attempt to expose
tango for neural network training for bot training." By IndianaOrz with FFCO,
September 2024 to January 2026. BN6 only. Last commits 2026-01-04 ("decent
fighting", "looking good mcts", "decent barrier").

**Base:** 2024 Tango (egui, the old `tango-pvp` crate). None of its Rust code
applies to current Tango directly.

**How it works:**

- Rust changes in `tango-pvp/src/game/bn6.rs` add `capture_frame_telemetry`,
  which reads BN6 memory each frame. Addresses, as they wrote them (**check
  every one against the current build**):

  | Field | Player 1 (obj0) | Player 2 (obj1) |
  | --- | --- | --- |
  | HP | `0x0203A9D4` | `0x0203AAAC` |
  | X | `0x0203AA4C` | `0x0203AB24` |
  | Y | `0x0203A9C4` | `0x0203AA9C` |
  | Charge | `0x0203409D` | `0x0203419D` |
  | Chip | `0x0203A9DA` | `0x0203AAB2` |
  | Emotion (in game) | `0x0203CE90` | `0x0203CE2C` |

  Local-console fields: emotion window `0x020352CC`, custom gauge
  `0x020352A1`, in-window flag `0x02035288` (255 = inside), beast selectable
  `0x0203664B`, cross window `0x020364C2`, chips selected count `0x020364C8`,
  chips visible count `0x020047D6`, menu index `0x020364C7`, cross index
  `0x020364DB`, selected indices base `0x02036508` (5 bytes), hand slots and
  codes at `0x0203CDB0` (16 bytes, alternating slot and code).

  Panel state: 18 addresses from `0x02039C06` in steps of `0x20` per column,
  with rows at `0x02039C06`, `0x02039D06`, `0x02039E06` (6 columns each).
  Panel owner is the next byte of each (`0x02039C07`, and so on).

  Note from their code: fields are read as obj0 and obj1 without regard to
  which side the local player is on. Their Python detects and swaps
  perspective afterwards, and many of their scripts fix swapped or flipped
  data. Build perspective handling into the observation from the start.
- A Python bridge: Tango sends frames and state to Python and receives inputs.
- Replays were converted to datasets (video, actions JSONL, static data).

**What they tried, in order:** image-based input prediction with damage
rewards; separate chip-select and battle models; critics trained with
reinforcement learning; fine-tuning NVIDIA's "NitroGen" game-playing model;
finally a hand-written Python BN6 battle simulator (`simulator/simcore/`,
`state.py`, `chips.py`, `mcts.py`) searched with MCTS. Their progression points
the same way as this plan: screen-image models struggled, and structured
state plus look-ahead did better. Using the real emulator for look-ahead
avoids their biggest cost, re-implementing the game.

The repo is research code across about a dozen branches (`MCTS`, `critic`,
`ng`, `plan` and others), with checked-in logs, debug images and a sample
replay.

## melonDS Android dual-screen fork

<https://github.com/SapphireRhodonite/melonDS-android>. Adds external and
dual display support to melonDS Android, with presets for which display gets
which screen. Reference files:

- `app/src/main/java/me/magnum/melonds/ui/emulator/render/ExternalPresentation.kt`
- `app/src/main/java/me/magnum/melonds/impl/layout/SecondaryDisplaySelector.kt`
- `app/src/main/java/me/magnum/melonds/ui/common/ExternalInfoPresentation.kt`

v0.0.5 fixed devices that report several internal displays by picking the
first truly external display. The Thor reports its top screen as primary.

## Sources

- <https://github.com/tangobattle/tango>
- <https://tango.n1gp.net/>, <https://lite.tango.n1gp.net/>
- <https://github.com/indianaorz/tango-ai>
- <https://retrogamecorps.com/2025/10/27/dual-screen-android-handheld-guide/>
- <https://retrohandhelds.gg/melonds-fork-brings-major-improvements-to-multi-display-functionality/>
- <https://liliputing.com/ayn-thor-is-dual-screen-android-handheld-game-console-with-oled-displays-and-qualcomm-snapdragon-inside/>
