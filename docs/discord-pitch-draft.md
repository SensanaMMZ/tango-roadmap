# Draft: matchmaking pitch for the N1GP Discord

Edit before posting. Keep it this short; details can follow in replies.

---

Hi! I'd like to contribute **public matchmaking** to Tango, and wanted to ask
whether you'd accept it upstream before I build it.

**What it is:** a "Find match" button. The server pairs two players in the
same game, patch and match type, then hands both a random session code.
Everything after that is the existing flow: signaling, lobby, compatibility
checks.

**Privacy:** public matches would always go through a TURN relay, so strangers
never see each other's IP. Rollback traffic is tiny, so relay cost stays low.

**Hidden rating:** a Glicko-2 rating per game family, stored on the server and
never shown, used only to pair people of similar skill. Identity works in
Tango Lite too: an anonymous WebCrypto device key by default, with optional
Discord login to keep your rating across devices. No client certificates.
Results come from both clients' reports; disagreements don't count and get
flagged, and leaving mid-match is a loss.

I'd write the queue service and the client side, following the existing
architecture. I'm using Claude Code for this, and would keep PRs small and
tested against the CONTRIBUTING checks.

Would you be open to this? If yes, would you want it on the existing
matchmaking server or as a separate service?
