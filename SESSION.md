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

## Decisions

- Order: **CPU bots → Android port → public matchmaking**.
- Work moves to the Windows machine. Native MSVC build for Tango, WSL2 for ML.
- Bots start with BN6.
- Code goes in a fork of `tangobattle/tango`; this repo stays planning-only
  and private.
- Pitch matchmaking **with** the hidden ranking on the N1GP Discord before
  building phase 3. See `docs/discord-pitch-draft.md`.

## Next step

Phase 1, milestone 0: follow `docs/windows-setup.md`, fork Tango, build it,
and run BN6 training mode. Then milestone 1: an `Opponent` trait replacing
`let dummy = 0;` in `tango-session/src/training.rs`.
