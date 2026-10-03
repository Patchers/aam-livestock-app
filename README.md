# AAM Livestock — app build

An installable PWA. Bottom tab navigation, full-screen detail views, filter chips,
bottom sheets, toasts, pull-to-refresh, saved-listings watchlist and offline caching.

## Files
- `index.html` — the whole app
- `manifest.json` — install metadata (name, icons, colours, standalone display)
- `sw.js` — service worker, caches the shell for offline use
- `icon-192.png` / `icon-512.png` / `icon-maskable.png` — home-screen icons

## Running it locally
The app loads Firebase as ES modules, which browsers block on `file://`. Serve it:
```
cd this-folder
python3 -m http.server 8080
```
Then open `http://localhost:8080` on your laptop, or your machine's LAN IP on your phone.

## Deploying
Push to `main`; GitHub Pages serves it from the root. Security rules are separate: paste
`firestore.rules` and `storage.rules` into the Firebase console whenever they change.

## Testing notes
- Farmers create an account on the Sell tab. Listings stay "In review" until approved.
- Admin is limited to the emails in `ADMIN_EMAILS` (index.html) **and** `firestore.rules`.
  The admin email must be verified — a link is sent on sign-up.
- On an empty database, the Admin tab offers **Load demo data** (sale yards, events, listings).
