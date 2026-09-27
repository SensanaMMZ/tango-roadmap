# Draft: matchmaking pitch for the N1GP Discord

Edit before posting. Keep it this short; details can follow in replies.

---

Hi! I'd like to contribute **public matchmaking for BN6** to Tango, and wanted
to ask whether you'd accept it upstream before I build it.

**Find match:** the server pairs two players in the same version (vanilla or a
mod's netplay group) and match type, then hands both a random session code.
Everything after that is the existing flow: signaling, lobby, compatibility
checks.

**Live board:** a screen showing how many people are searching and playing in
each BN6 version, so people can join whatever's active. You can fight a CPU
opponent while you wait (I'm building those too).

**Same compatibility rules:** pools are exactly the lobby's compatibility
tags plus match type, so the queue never pairs anyone the lobby would refuse.

**Privacy:** public matches always go through a TURN relay, so strangers never
see each other's IP.

**Hidden rating:** Glicko-2 per version, stored on the server and never shown,
used only to pair people of similar skill. Identity is an anonymous device key
on every platform (desktop, Tango Lite via WebCrypto, Android), with no
accounts and no client certificates.

**No manual reporting:** both clients already simulate both sides and trap the
game's own round results, so each sends a signed report automatically. Agreeing
reports count; disagreements don't and get flagged; leaving mid-match is a loss.

I'd write the queue service and client side, following the existing
architecture. I'm using Claude Code, and would keep PRs small and tested
against the CONTRIBUTING checks.

Would you be open to this? If yes, would you want it on the existing
matchmaking server or as a separate service?
