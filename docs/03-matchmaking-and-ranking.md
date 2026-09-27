# Phase 3: public matchmaking with a hidden rating

Goal: a "Find match" button that pairs players without typing a code, matched
by a hidden skill rating and by latency. A live board shows how many people are
searching and playing in each **version of BN6**, meaning vanilla plus each mod
in Tango's patch list. Players can join whichever is active, or fight a CPU
while they wait.

**Scope: Battle Network 6 only**, meaning all four cartridges and all BN6 mods.
**US and JP players share one pool.**

## BN6 in Tango today

Verified at upstream `84b2795` (`tango-gamesupport-bn6/src/lib.rs`,
`tango-library/src/patch/catalog.rs`, `tango-patch/src/tag.rs`) and in the
patch index (<https://github.com/tangobattle/patches>, surveyed 2026-09-27).

### Cartridges

| Tango family | Games (ROM code) | Region |
| --- | --- | --- |
| `bn6` | Cybeast Gregar (`BR5E`), Cybeast Falzar (`BR6E`) | US |
| `exe6` | Dennoujuu Glaga (`BR5J`), Dennoujuu Falzer (`BR6J`) | JP |

Match types: **Single, Triple, Random** (Random's rank select happens in-game).
Each client runs a shadow copy of the opponent's side, so **both players need
the opponent's ROM**. Legacy Collection owners have all four.

### How compatibility is decided

A pick resolves to a **tag** (`tango-patch/src/tag.rs`), and two players can
play only if their tags are equal:

- Unpatched game → `Vanilla { family }`.
- Mod marked `netplay = "vanilla"` (cosmetic: sound mods, translations) →
  same as unpatched.
- Mod marked `netplay = "group:<name>"` → `Group { family, group }`. Mods and
  versions sharing a group play each other.
- Mod with no `netplay` line → `Exact { family, patch, version }`. Only that
  exact mod version.

### Mods ("versions")

The patch index has **46 BN6 mods**: 29 built on the US ROMs (`bn6_*`) and 17
on the JP ROMs (`exe6_*`). Grouped by what can play what:

| Pool | Mods |
| --- | --- |
| Vanilla BN6 (US) | unpatched, `bn6_soundmod`, `bn6_soundmod_extend`, `bn6_Doronetwork` |
| Vanilla EXE6 (JP) | unpatched, `exe6_soundmod`, `exe6_soundmod_extend`, `exe6_cn`, `exe6_Doroexe` |
| `bingusbn6v1` | BingusBN6, BingusBN6+SoundMod |
| `bingusbn6v1infbeast` | BingusBN6 + Infinite Beast (with and without SoundMod) |
| `bingusexe6v1` | BingusEXE6, BingusEXE6+Soundmod, idealexe English |
| `bn6allstars_v1_1_0` | BN6 All-Stars + BBN6 (with and without SoundMod) |
| `exe6allstars_masters_v1_1_1` | EXE6 All-Stars Masters (JP and translated) |
| `exe6allstars_unseniors_v1_1_0` | EXE6 All-Stars Unseniors (JP and translated) |
| `exe6unseniors` | EXE6 Unseniors (JP and translated) |
| `exe6mb120` | idealexe English 120MB |
| `bn6_ba_crossover` | BA Crossover Black Ace, Red Joker (US Gregar only) |
| `bn6_ba_beta` | BA Star Force Black Ace, Red Joker Beta |
| `bn6_ba_lockon` | BA Star Force Black Ace, Red Joker Lock On |
| `Shift_0_1_0` | Shift |
| Exact version only | 40 Chip Folders, bn67, CyberBulliesBN6, BN6Daniel, DarkCross (BassCross, Star Force), BN6 Random Battle, Timaeus (Falzar), Infinite Beast, LDR Patch, Legend of NetBattles, Reverse Weaknesses, TwinLeaders, BA Crossover Black Ace (JP) |

Some mods only patch one cartridge (the BA mods need US Gregar; Timaeus needs
US Falzar).

## Merging US and JP into one pool

**The engine already supports it.** `tango-backend-mgba/src/backend.rs` looks
up each seat's ROM support from the family's seat table "because crossplay: a
Japanese cart links with an American one". BN6's table lists all four
cartridges.

**The lobby blocks it.** The vanilla tag is keyed by family ID, and `bn6` ≠
`exe6`, so a US player and a JP player get different tags and the lobby
refuses the pairing. (Test with a current build before relying on this; it's
read from the code, not observed.)

To merge them:

1. **Give the two families one netplay key.** For example, add a
   `netplay_family: "bn6"` field to `tango_gamesupport::Family`, set it on
   both `BN6_FAMILY` and `EXE6_FAMILY`, and build tags from it instead of
   `id`. Vanilla US and vanilla JP then share a tag, including their cosmetic
   mods.
2. **Verify it doesn't desync.** Run US vs JP matches in all three match types
   and compare both sides' input-log hashes and results. `sim_version` must
   match across the four seats (the backend requires it for crossplay
   siblings). Text length differences between regions are the likeliest
   source of timing drift, so watch chip select and round transitions.
3. **Mods merge only by their authors' choice.** Most mods have separate US
   and JP builds in different groups (`bingusbn6v1` vs `bingusexe6v1`), and the
   builds may really differ. Once the key is shared, a mod author can merge
   pools by giving both builds the same group name, after testing them against
   each other. Don't force it.

The rating follows the pool. With US and JP merged, vanilla BN6 has one rating
for everyone.

## How pairing works today

1. The player types a code. `tango-lobby`'s `LinkIdent::parse` turns it into
   `LinkIdent::Matchmaking(code)` (or a `/host`/`/connect` direct role).
2. `tango-net/src/connect.rs` connects to `wss://matchmaking.tango.n1gp.net`
   (`tango-library/src/config.rs`, `DEFAULT_MATCHMAKING_ENDPOINT`) with the
   code as the session ID. Protocol: `tangobattle/tango-signaling`,
   `src/proto/signaling.proto`.
3. The server pairs the two sockets sharing that ID and relays the WebRTC
   offer, answer and ICE candidates. It also hands out ICE (STUN/TURN) servers.
4. The players connect directly. The lobby exchanges `Settings`
   (`tango-net-protocol/src/control.rs`: nickname, match type, game and
   variant, patch, `sim_version`, blind setup) and refuses incompatible
   pairings.

The current server's source is not in the tangobattle GitHub org. The 2024
version (`tango-signaling-server`, about 500 lines of Rust, a `HashMap` of
session IDs) survives in old forks such as `indianaorz/tango-ai`.

**So a public queue only has to choose the session code.** Everything after
step 2 already works.

## Player identity: anonymous device key, on every platform

Every client generates a key pair on first launch. The public key's
fingerprint is the player ID. To prove who it is, the client signs a random
challenge from the server. No account, no login, no client certificates, so
it works in browsers.

Use **ECDSA P-256** everywhere. It's the one algorithm that's non-extractable
in every browser's WebCrypto and hardware-backed in Android Keystore.

| Platform | Where the key lives |
| --- | --- |
| Desktop (Windows, macOS, Linux) | Private key file in Tango's config directory, next to `config.json`. The OS keychain is optional hardening later |
| Browser (Tango Lite) | WebCrypto key with `extractable: false`, stored in IndexedDB. Page scripts can't read the private key bytes |
| Android | Android Keystore (hardware-backed where available) |

**Consequences to design for:**

- **Losing the key means a new player.** Reinstalling, clearing browser data or
  switching devices starts a fresh, provisional rating. Offer:
  - **Export and import** on desktop and Android: a backup file of the key.
  - **Link a device:** the old device signs a short-lived transfer code that
    the new device redeems. The server then maps both keys to one player.
    This is the only way to move a browser identity, since its key can't be
    exported.
- **Free resets allow smurfing.** A new key is a new player. Glicko-2 and
  OpenSkill give new players high uncertainty, so a strong player's fresh key
  climbs to its real level within a handful of matches. The rating is hidden,
  so there's little to gain from resetting.
- **Discord linking can come later** as an option. It isn't needed for launch.

## Reporting wins and losses automatically

Tango already knows the result without asking anyone:

- **Both clients simulate both sides.** Each client runs both players' games
  in lockstep from the same confirmed inputs, so each one independently
  reaches the same result.
- **The game itself says who won.** BN6's hooks trap the game's own
  round-result code (`round_end_set_win`, `round_end_set_loss` and the
  damage-judge variants in `tango-gamesupport-bn6/src/pvp.rs`). They feed
  `telemetry::Event::RoundEnded { outcome }` (`P0Win`, `P1Win`, `Draw`) and
  `Event::MatchEnded`.
- **The app already gets stats at the end.** `tango-session`'s `StatsSink`
  receives each finished match's `MatchStats`, including its rounds.

Automatic reporting needs these additions:

1. When the queue pairs two players, the server issues a **match ID** along
   with the session code.
2. On `MatchEnded`, each client sends a **signed result report** with no
   player action:
   - match ID
   - per-round outcomes and the final winner
   - a hash of the confirmed input log (the replay's inputs)
   - final tick count

   It's signed with the player's device key.
3. The server compares the two reports. The simulation is deterministic, so
   honest clients agree exactly, hash included.
   - **Both agree:** apply the rating change.
   - **They disagree:** no rating change, and flag both players. Repeat
     disagreements point to a modified client.
4. **Disconnects:**
   - Tango already reconnects automatically. Give it a grace period before
     deciding anything.
   - The session supervisor already tells an orderly shutdown (`Goodbye`)
     apart from a dropped connection.
   - If only one client reports, and it reports that the opponent dropped
     mid-match, the one who left takes the loss.
   - If neither reports (a server outage, or both dropped), nothing is
     recorded.
5. **Disputes:** a replay holds inputs, not ROMs, so a client can upload it on
   request. A moderator with their own ROMs can replay it to see who is right.
   The server never needs the ROMs.

A modified client can still lie. Two-sided agreement, input-log hashes,
flagging, and a hidden rating (little incentive to cheat) keep that
manageable for a community game.

## Live board: who's searching and who's playing

One row per **version**: vanilla BN6 (US and JP together) and each mod pool,
with counts per match type.

```
Version                           Single     Triple     Random
Vanilla BN6 (US + JP)             3 / 4      1 / 6      0 / 1
BingusBN6                         2 / 2      4 / 8      -
BN6 All-Stars + BBN6              0 / 2      1 / 0      -
EXE6 Unseniors                    0 / 0      1 / 2      -
LDR Patch 1.5.8                   0 / 0      0 / 2      -
▸ 38 more versions with no one online
                         searching / playing   ·   Playing CPU while waiting: 5
```

- A row is a **pool** (a compatibility tag), not a single mod. It's labelled
  with the mod's title, and its mods are listed when expanded.
- Sort active pools first, and collapse the rest. With 46 mods, most rows will
  be empty most of the time.
- Clicking a cell queues for that pool and match type. If the player doesn't
  have the mod yet, Tango's existing patch download runs first. If they lack a
  cartridge the pool needs (the BA mods need US Gregar), the cell says so.
- **Data source:** the matchmaking server. Searching counts come from its
  queue. Playing counts come from pairs it created, ended by result reports or
  a timeout. The board screen gets counts over the same websocket, pushed
  every few seconds or on change.
- **Optional:** clients could also report private-code and direct matches, so
  "playing" counts everyone. That's opt-in, since the server doesn't see those
  games otherwise.

## Fighting a bot while you wait

- Queueing and playing are independent. A player can queue, then start a
  "vs CPU" match (phase 1) while the queue runs in the background.
- On a pairing, show "Match found" with accept and decline over the bot
  match. Accepting ends the bot match; bot matches never affect the rating.
- The board shows how many people are playing a CPU while waiting. That shows
  there's activity even when the queue is short.

## Queue service

The client sends a queue request:

- pool: the compatibility tag (vanilla BN6, a mod group, or an exact mod
  version) and match type
- the cartridge and mod it will play
- `sim_version` and protocol version
- rough region or ping to the server
- player key fingerprint and challenge signature

The server pairs within a pool by rating and ping, widening the rating window
over time. On a pair it sends both players the session code and match ID.
Everything after that is the existing flow.

**Client:** "Find match", the board, accept and decline, auto-ready,
timeouts, penalties for declines and dodges.

## Rating

- Glicko-2 or OpenSkill, stored server-side, keyed by player ID.
- A separate rating per pool (vanilla BN6 with US and JP merged, and each mod
  pool). Mods change the game enough that skill doesn't fully carry over.
  Seed a player's first rating in a new pool from their vanilla rating.
- Possibly separate per match type.
- Never shown in the client. Used only for pairing.
- Folder and NaviCust quality matter a lot in BN, so the rating measures save
  strength plus skill. Blind setup (already in `Settings`) or a
  standard-folder mode can help if that's a problem.

## Risks

- **Small player base.** Pools split by mod and match type (merging US and JP
  helps). The
  board and bot-while-waiting both exist to make waits bearable. Widen
  matching quickly.
- **IP exposure.** With codes you connect to people you chose. With a public
  queue, strangers see your IP. Force TURN relay for public matches.
  Rollback traffic is tiny, so relay bandwidth is cheap. The old server had a
  Cloudflare ICE config backend for this.
- **Splitting the community.** A fork-run server can't pair with official
  Tango players. Get this accepted upstream rather than running a rival
  server. It also lets Android players meet PC players.

## Effort

Queue, board, and client flow: a few weeks. Device keys, automatic reports,
disputes and rating: a few more. Then ongoing hosting and moderation.

## Pitching it upstream

Upstream has GitHub Issues and Discussions turned off. Post on the **N1GP
Discord** (<https://discord.n1gp.net>), which runs the matchmaking server.
`bigfarts` wrote almost all of the code (about 5,150 commits); `ArthurCose` is
second (about 120). Draft: [discord-pitch-draft.md](discord-pitch-draft.md).

Lead with:

1. The queue only hands both players a random session code; everything after
   pairing already works.
2. Public matches go through a TURN relay so strangers never see each other's IP.
3. One pool for US and JP. The engine already supports crossplay; only the
   lobby's compatibility tag separates them.
4. A hidden rating, stored server-side per pool. Identity is an anonymous
   device key on every platform, including Tango Lite, so it doesn't bring
   back the client-certificate problem.
5. Results are reported automatically, since both clients already simulate
   both sides and trap the game's own result code.
6. A live board of searching and playing counts per BN6 version (vanilla and
   each mod pool).

Ask whether they'd accept it before building, and offer to write it.

### How upstream uses AI

There's no written policy on AI contributions (no mention in `CONTRIBUTING.md`,
no `AGENTS.md` or `CLAUDE.md`). The maintainer uses Claude Code heavily:

- 1,387 of the most recent ~3,000 upstream commits carry a
  `Co-Authored-By: Claude` trailer, all by `bigfarts`, from 2026-05-12 to
  2026-08-10.
- 1,088 of the 1,281 commits since 2026-06-01 have it. The ~60 commits after
  2026-08-10 have no trailer, so they may simply have stopped adding it.

So AI-assisted work is normal there. That doesn't guarantee they'll take
AI-written pull requests from others. Say you used Claude Code, keep PRs
focused, run the upstream checks, and follow their style (dense comments
explaining why).
