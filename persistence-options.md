# Tournament Persistence Options

Currently all state is in `localStorage` — survives refresh on the same device/browser, gone everywhere else.

---

## Option 1: Export / Import JSON
A "Save" button downloads (or copies to clipboard) the tournament as a JSON blob. A "Load" button reads it back.

- No infrastructure
- Good for backup and recovery ("I lost my phone")
- Bad for live sync — someone has to manually send the file around

---

## Option 2: Shareable URL
Encode the full tournament state as base64 in the URL hash (e.g. `.../#eyJwbGF5...`). Anyone with the link sees the exact state at that moment.

- No server
- Easy to share via text or QR code
- URL doesn't update automatically — you re-share after big changes
- Pairs well with Option 1

---

## Option 3: JSONBin / npoint (paste API)
Services like [jsonbin.io](https://jsonbin.io) or [npoint.io](https://npoint.io) provide a free hosted JSON file via API. The app generates a short ID that the group shares. Anyone enters that ID to load the same state.

- No server to run
- Depends on a third-party free service
- Near-real-time sync with a manual refresh

---

## Option 4: Cloudflare Worker + KV
A tiny Worker (< 20 lines) acts as a relay — PUT saves tournament JSON to KV storage, GET retrieves it by ID. Free tier is generous.

- You own the data, no third-party dependency
- More setup than Option 3
- Scales to real-time sync if needed later

---

## Recommendation

**Start with Option 2 + 1** (shareable URL + export/import). Covers the "everyone can see it" case with zero infrastructure and the backup/recovery case. If live sync proves important, Option 3 or 4 can layer on top.
