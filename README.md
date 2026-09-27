# Tango roadmap: CPU bots, Android port, public matchmaking

Planning repo for extending [Tango](https://github.com/tangobattle/tango), the
rollback-netplay frontend for Mega Man Battle Network, with three features,
built in this order:

1. **CPU opponents** that players can fight offline, starting with BN6.
2. **An Android port** with full support for the AYN Thor's two screens,
   including the DS game (BN5 Double Team DS) and a bottom-screen layout for the
   GBA games.
3. **Public matchmaking**: a queue that pairs players without typing a code,
   with a hidden skill rating used only for pairing.

Nothing here is code yet. These docs capture the research from a first Claude
Code session (2026-09-27) so work can resume on the Windows training machine.

## Start here

| Doc | What's in it |
| --- | --- |
| [CLAUDE.md](CLAUDE.md) | Context Claude Code loads automatically in this repo |
| [docs/windows-setup.md](docs/windows-setup.md) | Toolchain for building Tango and training bots on Windows |
| [docs/01-cpu-bots.md](docs/01-cpu-bots.md) | Phase 1: bot design, where it plugs into Tango, milestones |
| [docs/02-android-port.md](docs/02-android-port.md) | Phase 2: Android host, native cross-compile, Thor dual screen |
| [docs/03-matchmaking-and-ranking.md](docs/03-matchmaking-and-ranking.md) | Phase 3: queue server, identity, results, rating, pitching upstream |
| [docs/discord-pitch-draft.md](docs/discord-pitch-draft.md) | Draft post pitching matchmaking and ranking to upstream |
| [docs/tango-codebase-notes.md](docs/tango-codebase-notes.md) | How current Tango is laid out, with file pointers |
| [docs/prior-work.md](docs/prior-work.md) | tango-ai, Tango Lite, melonDS Android dual-screen fork |
| [SESSION.md](SESSION.md) | What was asked and decided in the first session |

## Hardware

- **Windows machine (primary):** RTX 4090 (24 GB), 16 cores / 32 threads, 64 GB RAM.
  Covers every bot approach, including self-play training. No cloud rental needed.
- **Linux laptop (secondary):** Core Ultra 7 258V, GTX 1660 Super (6 GB), 30 GB RAM.
  Fine for coding and the rules-based bot.

## Upstream facts that shape the plan

- Tango upstream is <https://github.com/tangobattle/tango> (GPL-3.0). It was last
  surveyed at commit `84b2795` (2026-09-23).
- Upstream **has GitHub Issues and Discussions turned off**. Suggestions go to the
  N1GP Discord: <https://discord.n1gp.net>. The main author is `bigfarts`.
- Forks of a public repo are public on GitHub. This planning repo is private;
  the code fork of Tango will not be.
