# Rummikub Clock — Feature Plan

## Constraints

- Todd deploys via GitHub Pages (pure static hosting — no server-side code, no env vars)
- Todd is non-technical; he should not need to touch any infrastructure
- Chad owns and maintains the Vercel backend; it's invisible to Todd
- No API keys in the repo (public repo, keys get auto-revoked)

## Working with Todd's Repo

Chad maintains a fork of Todd's repo. Todd is actively developing independently and doesn't merge PRs promptly. Workflow:
- Keep PRs small and focused so they're easy for Todd to review
- Rebase the fork onto upstream regularly (low cost — our changes are isolated)
- Try to loop Todd in before he pushes big batches of changes
- Forking (not collaborating directly in Todd's repo) is the right structure — gives Chad an independent GitHub Pages deployment and keeps experimental work separate

---

## Shipped

### Scoreboard: one winner per game
Each tally box now represents a specific game, not a freeform check. Only one player can hold the win per game column. Changes:
- Radio-button behavior per column (tapping clears other players' marks for that game)
- Prompt before reassigning a game win to a different player
- Prompt before removing a recorded win
- "GAME 1–10" header row above the tally grid
- Alternating column shading to reinforce vertical game grouping
- Filled boxes show the player's accent color

### Audio fix: screen lock / tab background on iOS
iOS Safari suspends the AudioContext when the screen locks or the tab backgrounds. During a real game the phone sits idle between turns long enough to trigger this.
- **Screen Wake Lock** — acquired on `startTimer`, released on pause/stop, re-requested on `visibilitychange` when running. Prevents the screen from locking mid-turn. Degrades silently on unsupported browsers.
- **AudioContext resume on visibility change** — calls `getAudioCtx()` on page restore, which re-arms the suspended context.

---

## Planned

### Infrastructure: Vercel Serverless Functions (Chad owns)

A small Vercel project acts as a backend proxy. It holds all secrets as Vercel environment variables and exposes a simple REST API to the static app. Todd deploys to GitHub Pages as normal; the Vercel URL is just a hardcoded constant in the code.

The Vercel project handles two things:

1. **Vision requests** — proxies image data to Claude Vision API, returns tile values
2. **Tournament state** — reads/writes JSON blobs to Vercel KV (Upstash Redis) by short ID

**Repo structure**
The Vercel backend lives in its own GitHub repo (e.g. `rummikub-api`), separate from the fork. It's small enough that Todd could absorb it into the main app later if he ever wanted to own the infra. The only footprint in the fork is a single hardcoded Vercel URL constant.

**Security**
Risk is effectively zero — the URL isn't publicized and the audience is a small group of Rummikub players. Light hardening:
- **Shared secret** — app sends a hardcoded key in each request header; Vercel rejects anything without it
- **Origin check** — only accept requests from `toddjmitch.github.io` and `chadallen.github.io`
- **Rate limiting** — Vercel built-in, e.g. 50 requests/hour

**One-time setup**
1. Create a new Vercel project (separate from any existing projects)
2. Add `ANTHROPIC_API_KEY` as an environment variable in Vercel
3. Add Vercel KV storage to the project via the Vercel dashboard
4. Deploy two API routes: `/api/vision` and `/api/tournament`
5. Paste the Vercel project URL as a constant in the app code

### Feature 1: Vision-Based Scoring

At game end, each losing player photographs the tiles remaining on their rack. The app sends the image to the Vercel backend, which calls Claude Vision and returns the list of tile values. The app sums them and enters the penalty score automatically.

**User flow**
1. Tap "Score round" after someone goes out
2. Each losing player taps "Scan rack" and points their camera
3. App displays detected tiles and total — player confirms or corrects
4. Score is recorded

**Notes**
- Jokers count as 30 pts — vision model should detect them
- Rummikub tiles are clean and high-contrast; vision accuracy should be high
- Manual override always available if scan fails

### Feature 2: Tournament Persistence & Sharing

Tournament state currently lives only in localStorage on one device. If that device goes away, the tournament is lost.

**Approach**
- The Vercel backend stores tournament JSON in Vercel KV
- On "Save", the app POSTs state and gets back a short ID (e.g. `abc123`)
- Anyone with that ID can load the tournament from any device by entering it
- Could also encode as a shareable URL: `.../?t=abc123`

**Also worth adding regardless**
- Export to JSON (download file) — zero-dependency backup
- Import from JSON — recovery if Vercel is unavailable

**Notes**
- State doesn't need to be real-time; a manual "sync" button is fine
- The short ID could be displayed as a QR code for easy sharing at the table
