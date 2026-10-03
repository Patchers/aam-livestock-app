# AAM Livestock — app build

An installable PWA. Bottom tab navigation, full-screen detail views, filter chips,
bottom sheets, toasts, pull-to-refresh, saved-listings watchlist and offline caching.

## Files
- `index.html` — the whole app
- `manifest.json` — install metadata (name, icons, colours, standalone display)
- `sw.js` — service worker, caches the shell for offline use
- `icon-192.png` / `icon-512.png` / `icon-maskable.png` — home-screen icons

## Demoing it now
Open `index.html` in a browser. Everything works except install and offline —
those need the files served over HTTP(S), which is a browser security rule, not a bug.

To test those locally:
```
cd this-folder
python3 -m http.server 8080
```
Then open `http://localhost:8080` on your laptop, or your machine's LAN IP on your phone.

## Deploying
Push all files to a GitHub repo, then **Settings → Pages → Deploy from branch → main → / (root)**.
Once live over HTTPS, phones will offer "Add to Home Screen" and it opens fullscreen
with no browser bars.

## Demo notes
- Any email and password logs you in on both Sell and Admin.
- Data resets on refresh — handy for re-running a demo.
- Submit a listing under Sell, then approve it under Admin to show the full loop.
