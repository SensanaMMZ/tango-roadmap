# Session log: 2026-09-27 (Claude Code, Linux laptop)

A condensed record of the first planning session, so a new session doesn't
repeat it. Details are in `docs/`.

## Questions and answers

1. **"What would I need to do to make an Android port of Tango and have it
   fully support the AYN Thor dual screen?"**
   Write a new Android frontend (Kotlin plus a Rust JNI crate modelled on
   `tango-lite-web`), not a port of the iced desktop app. Cross-compile mGBA,
   melonDS, libdatachannel with `cargo-ndk`. Use a `Presentation` on the
   Thor's bottom display for the DS touch screen and for GBA match info.
   → `docs/02-android-port.md`

2. **"Also look for other attempts online."**
   No Android port exists. Closest things: Tango Lite (official browser
   build, one screen), and the melonDS Android dual-screen fork as a
   reference for second-display code. → `docs/prior-work.md`

3. **"What is tango-ai?"**
   A hobbyist fork (IndianaOrz, FFCO) experimenting with BN6 bots from
   September 2024 to January 2026, built on 2024 Tango. Moved from
   screen-image models to a hand-written simulator with MCTS.
   → `docs/prior-work.md`

4. **"What would it take to make competent CPU players?"**
   Training mode already runs a local battle against a do-nothing dummy;
   replace its input with a bot. Extend the existing per-game telemetry for
   observations. Start with rules plus look-ahead using the real emulator,
   then imitation learning from replays. → `docs/01-cpu-bots.md`

5. **"Public matchmaking without codes, plus a hidden ranking?"**
   A queue server that hands both players a random session code; the rest of
   the pipeline already works. Needs identity, result reporting, Glicko-2 or
   OpenSkill per game family, TURN relay for IP privacy.
   → `docs/03-matchmaking-and-ranking.md`

6. **"Where can I make these suggestions?"**
   Upstream has GitHub Issues and Discussions off. Use the N1GP Discord
   (<https://discord.n1gp.net>). Main author: `bigfarts`.

7. **"What GPU would I need for the CPU players?"** and **"Would players need
   it too?"**
   Only training needs serious hardware; CPU cores matter as much as the GPU
   for self-play. Players only run the finished bot, which needs nothing extra
   for rules or small models.

8. **Hardware:** the Windows machine (RTX 4090, 16C/32T, 64 GB) covers every
   approach. It becomes the primary machine.

9. **"Include the ranking in the ask; couldn't identity be stored server-side
   for browser players?"**
   Yes. The rating is server-side regardless. Identity for browsers works with
   Discord OAuth or a WebCrypto device key signing a server challenge; only
   mutual TLS was a browser problem. The pitch now includes ranking.

10. **"How do they feel about AI usage?"**
    No written policy, but the maintainer uses Claude Code heavily: 1,387 of
    the last ~3,000 upstream commits have a `Co-Authored-By: Claude` trailer.
    Disclose AI use and keep PRs focused and tested.

11. **"Anonymous key as the default on all platforms."**
    Done: ECDSA P-256 device key on desktop (config dir), Tango Lite
    (non-extractable WebCrypto key in IndexedDB) and Android (Keystore).
    Export/import and "link a device" cover lost keys; Discord linking is
    optional and later.

12. **"How can wins and losses be reported without user input?"**
    Both clients simulate both sides and trap the game's own round-result
    code, so each already knows the result. At match end each client sends a
    signed report (match ID, round outcomes, input-log hash). Agreeing
    reports count; disagreements are flagged; leaving mid-match is a loss.

13. **"Scope bots and matchmaking to BN6 only."** And: **"By version I mean
    Tango's modded versions of BN6."**
    Tango's patch index has 46 BN6 mods (29 US-based, 17 JP-based). The board
    and the rating are per pool: vanilla, or a mod's netplay group, or an exact
    mod version.

14. **"UI showing how many users are searching or in matches per version, and
    bots while waiting."**
    Live board fed by the matchmaking server over websocket; queue runs in the
    background during a CPU match; "match found" interrupts it.

15. **"US and JP should not use different pools."** and **"What does 'the
    lobby's compatibility tag distinguishes by family ID' mean?"**
    Tango files the US games under family `bn6` and the JP games under family
    `exe6`. Before a match, each player's pick becomes a compatibility tag
    built from that family ID, and different tags are refused. So vanilla US
    and vanilla JP can't match today, although the emulation engine was built
    to support it.

16. **"If they distinguish compatibility like that, then follow it."**
    Decision reversed: pools use Tango's compatibility tags unchanged, so US
    and JP stay separate. No compatibility changes proposed upstream.

## Decisions

- Order: **CPU bots → Android port → public matchmaking**.
- Work moves to the Windows machine. Native MSVC build for Tango, WSL2 for ML.
- Bots and matchmaking cover **BN6 only**: all four cartridges and its mods.
- **Pools follow Tango's compatibility tags unchanged**; US and JP stay
  separate.
- **Anonymous device key** is the default identity everywhere.
- **Results are reported automatically.**
- A **live board** of searching and playing counts per version, with CPU
  matches while waiting.
- Code goes in a fork of `tangobattle/tango`; this repo stays planning-only
  and private.
- Pitch matchmaking **with** the hidden ranking on the N1GP Discord before
  building phase 3. See `docs/discord-pitch-draft.md`.

## Next step

Phase 1, milestone 0: follow `docs/windows-setup.md`, fork Tango, build it,
and run BN6 training mode. Then milestone 1: an `Opponent` trait replacing
`let dummy = 0;` in `tango-session/src/training.rs`.
