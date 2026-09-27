# Phase 3: public matchmaking with a hidden rating

Goal: a "Find match" button that pairs players without typing a code, matched
by a hidden skill rating and by latency.

## How pairing works today

1. The player types a code. `tango-lobby`'s `LinkIdent::parse` turns it into
   `LinkIdent::Matchmaking(code)` (or a `/host`/`/connect` direct role).
2. `tango-net/src/connect.rs` connects to `wss://matchmaking.tango.n1gp.net`
   (`tango-library/src/config.rs`, `DEFAULT_MATCHMAKING_ENDPOINT`) with the code
   as the session ID. Protocol: `tangobattle/tango-signaling`,
   `src/proto/signaling.proto`.
3. The server pairs the two sockets sharing that ID and relays the WebRTC offer,
   answer and ICE candidates. It also hands out ICE (STUN/TURN) servers.
4. The players connect directly. The lobby exchanges `Settings`
   (`tango-net-protocol/src/control.rs`: nickname, match type, game and
   variant, patch, `sim_version`, blind setup) and refuses incompatible
   pairings.

The current server's source is not in the tangobattle GitHub org. The 2024
version (`tango-signaling-server`, about 500 lines of Rust, a `HashMap` of
session IDs) survives in old forks such as `indianaorz/tango-ai`.

**So a public queue only has to choose the session code.** Everything after
step 2 already works.

## What to build

### Queue service

The client sends a queue request:

- compatibility key: game family and variant, patch and version,
  `sim_version`, protocol version, match type
- rough region or ping to the server
- player ID

The server only pairs players whose compatibility key matches exactly. Within
that group it pairs by rating and ping, widening the rating window the longer
someone waits. On a pair, it generates a random session ID and sends it to
both players.

### Client

- "Find match" entry point next to typed codes and direct connect
- "Match found" screen with accept and decline
- automatic ready-up, timeouts, penalties for declines and dodges
- queue population display

### Player identity (works for browser players too)

The rating itself always lives **server-side**, in the matchmaking server's
database, keyed by player ID. The only question is how the server knows who is
connecting.

Tango used to present an install certificate over mutual TLS. It was removed
because browsers can't present client certificates; the field is reserved in
`signaling.proto`. That only rules out mutual TLS. Two schemes work the same
in the desktop app, Tango Lite and Android:

- **Login with Discord (OAuth).** The server issues a session token after
  login. The client stores it (browser storage, app config) and sends it when
  it queues. The community already lives on Discord, and this makes new
  accounts cost something, which limits rating resets.
- **Anonymous device key, signed challenge.** The client generates a key pair
  once (in browsers, a non-extractable WebCrypto key kept in IndexedDB) and
  signs a nonce the server sends. The public key's fingerprint is the player
  ID. No login needed, but clearing browser data or reinstalling makes a new
  player, so it's easy to reset a rating.

Best of both: start everyone on a device key so queueing needs no login, and
let players link Discord to keep their rating across devices and browsers.

### Result reporting

The match is peer-to-peer, so the server never sees it.

- Both clients report the result. Agreeing reports change ratings;
  disagreeing ones change nothing and get flagged.
- A disconnect mid-match counts as a loss for whoever left. The session's
  supervisor already distinguishes a dropped connection from an orderly
  shutdown.
- Replays are deterministic and could settle disputes by server-side
  re-simulation, but the server would need the ROMs. That's a legal problem,
  so rely on agreeing reports and flag repeat offenders.

### Rating

- Glicko-2 or OpenSkill. Both track uncertainty, so new players settle fast.
- A separate rating per game family (BN6 skill doesn't carry to BN3), possibly
  per match type.
- "Hidden" means the client never displays it.
- Folder and NaviCust quality matter a lot in BN, so the rating measures save
  strength plus skill. Blind-setup or standard-folder modes help if that's a
  problem.

## Risks

- **Small player base.** Ten games, patch versions and regions split the queue.
  Show queue counts, widen matching quickly, and offer a CPU match while
  waiting (phase 1 pays off here).
- **IP exposure.** With codes you connect to people you chose. With a public
  queue, strangers see your IP. Force TURN relay for public matches. Rollback
  traffic is tiny, so relay bandwidth is cheap. The old server had a
  Cloudflare ICE config backend for this.
- **Splitting the community.** A fork-run server can't pair with official
  Tango players. Get this accepted upstream rather than running a rival server.
  It also lets Android players meet PC players.

## Effort

Queue service plus client flow: a few weeks. Accounts, result reporting and
disputes: a few more. Then ongoing hosting and moderation.

## Pitching it upstream

Upstream has GitHub Issues and Discussions turned off. Post on the **N1GP
Discord** (<https://discord.n1gp.net>), which runs the matchmaking server.
`bigfarts` wrote almost all of the code (about 5,150 commits); `ArthurCose` is
second (about 120).

Keep the first post short. Include the ranking; it's the main reason a
public queue beats typed codes. Lead with:

1. The queue only hands both players a random session code; everything after
   pairing already works.
2. Public matches go through a TURN relay so strangers never see each other's IP.
3. A hidden rating stored server-side, per game family, used only for pairing.
   Identity works in browsers (Discord login or a WebCrypto device key), so it
   doesn't bring back the client-certificate problem they removed.

Expect questions on identity and result reporting; have the answers from this
doc ready. Ask whether they'd accept it before building, and offer to write it.

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
