# CONTEXT — AAM Livestock

Read this before changing anything. It captures decisions already made so they don't get re-litigated.

## What this is
An installable PWA for a livestock auctioneering business in the KZN Midlands, South Africa.
Three audiences in one app: buyers browsing sales and stock, farmers listing animals,
and the office approving those listings.

Wired to Firebase (Auth, Firestore, Storage). Currently in a **testing phase**: the client's staff
try it out while the developer stays the only admin. Going live is configuration, not a rebuild —
swap the admin emails, clear test data, and transfer project ownership/billing.

## Files
- `index.html` — the entire app: markup, CSS, and JS in one file
- `manifest.json` — PWA install metadata
- `sw.js` — service worker, caches the app shell (same-origin + Firebase SDK only)
- `firestore.rules`, `storage.rules` — **source of truth** for security rules; paste into the
  Firebase console (no CLI deploy is set up)
- `icon-*.png` — home screen icons

## Architecture decisions (settled — don't revisit)
- **Hosting:** GitHub Pages. Static only. All paths are relative so it works in a subdirectory.
- **Backend:** Firebase — Auth (email/password), Firestore, Cloud Storage.
- **No build step.** Plain HTML/CSS/JS, no npm, no bundler, no framework. Keep it that way —
  the client needs to be able to hand this to any developer later.
- **Single file** for the app. Don't split into modules; it's deliberate.
- Firebase SDK via CDN ESM imports (`https://www.gstatic.com/firebasejs/…`), not npm.

## Data model
```
listings/{id}
  lot, breed, type, wt, price, loc, desc,
  qty,                         // head count for a herd lot (>= 2); null = single animal.
                               // For a herd, wt is average weight and price is for the whole lot
  who, tel,                    // seller name + phone
  img,                         // Cloud Storage download URL
  status: "pending"|"live"|"rejected",
  ownerUid, createdAt

events/{id}
  title, cat, date, start, end,   // cat is "Standard Sale" | "Production Sale"
  ven,                            // venue doc id
  who, tel, em, note, img,        // field officer contact
  createdAt

venues/{id}
  name, town, contact, map, note  // map is a Google Maps URL
```

## Business rules
- New listings are created with `status: "pending"`. They do **not** appear publicly.
- The Market tab reads only `status == "live"`.
- The Sell tab's "Your listings" reads `ownerUid == currentUser.uid`, all statuses,
  showing a pill: In review / Live / Declined.
- Only admins can change `status`, or write to `events` and `venues`.
- Admin is determined by an `ADMIN_EMAILS` allowlist in the client **and** enforced in
  Firestore rules. The client check is UX only; the rules are the actual security.
- The Sales tab shows only future events (`date >= today`), sorted ascending.
- Past events still show in the admin Events list, greyed out.

## Firebase project
Project ID: `aam-livestock`. Firestore **Standard** edition, region `europe-west1`.
Plan is **Blaze** (pay-as-you-go, on the $300 Google Cloud credit) — Storage requires
this now, Firestore/Auth would work on the free Spark plan but the project is on Blaze
regardless. A budget alert is set; don't let usage patterns in generated code ignore that
(e.g. no polling loops, no unbounded `onSnapshot` without cleanup on unmount).

**Storage bucket is in `us-central1`, not `europe-west1`.** This is deliberate — it's one
of the three regions covered by Cloud Storage's Always Free tier, so uploads and downloads
at demo volume cost nothing. Firestore stays in europe-west1. Different regions for
different services is intentional, not a mistake to "fix."

```js
const firebaseConfig = {
  apiKey: "AIzaSyDgM54QWo4U8eHO0pEjP_52lnE9JWGgPEc",
  authDomain: "aam-livestock.firebaseapp.com",
  projectId: "aam-livestock",
  storageBucket: "aam-livestock.firebasestorage.app",
  messagingSenderId: "513990494263",
  appId: "1:513990494263:web:14736bab75f3b28b3997b2"
};
```

Two things about that snippet as Firebase hands it to you:

- It uses npm-style imports (`from "firebase/app"`). **There is no build step here.** Rewrite
  them as CDN ESM URLs — `https://www.gstatic.com/firebasejs/<version>/firebase-app.js` and
  likewise for `firebase-auth.js`, `firebase-firestore.js`, `firebase-storage.js`. Check the
  current SDK version rather than assuming; they move.
- `storageBucket` ends in `.firebasestorage.app`, not `.appspot.com`. That is correct for
  projects created recently. Do not "fix" it to `appspot.com` — older tutorials say that and
  it will silently break uploads.

The `apiKey` is a public identifier, not a credential. It ships in the client of every
Firebase web app. Security is entirely in the rules below.

## Firestore rules — `firestore.rules`
Production mode defaults to deny-all. Skip publishing the rules and the app renders perfectly and
shows nothing, which looks like a data bug and isn't.

Beyond the original design, the rules file adds:
- `isAdmin()` also requires `email_verified` — otherwise anyone could sign up with an admin
  address first. Admin = `johnchutton@gmail.com` during testing.
- `validListing()` type-checks `price`, `wt`, `qty` and restricts `img` to Storage URLs, because
  those values are rendered into the page.
- Admins may create listings in any status (used by the one-off "Load demo data" button).

**The gotcha that catches everyone:** rules do not filter queries, they validate them.
`getDocs(collection(db,'listings'))` fails outright even though some documents are public.
The query must itself constrain to what the rules allow:
`query(collection(db,'listings'), where('status','==','live'))`.

## Storage rules — `storage.rules`
Uploads go to `listings/{uid}/…`, images only, under 3 MB.
The 3 MB ceiling means client-side compression is mandatory, not optional — phone photos
routinely exceed it. Resize to ~1600px on the long edge and re-encode as JPEG before upload.

## What was wired up (done)
1. Firebase init + config at the top of the script block.
2. Auth: replace the fake login on the Sell and Admin tabs with real
   `signInWithEmailAndPassword` / `createUserWithEmailAndPassword`.
   Use `onAuthStateChanged` to restore sessions on load.
3. Replace the `venues`, `events`, `stock` arrays with Firestore reads.
   Prefer `onSnapshot` so the admin approving a listing updates the Market tab live.
4. `submitListing()` — upload the photo to Storage first, then `addDoc` with the URL.
   Currently it base64s the image into memory; that must not go into Firestore
   (1 MB doc limit). Compress client-side before upload; phone photos are 3–8 MB.
5. `setSt()`, `saveEv()`, `delEv()`, `saveVen()`, `delVen()` — swap array mutation for
   `updateDoc` / `addDoc` / `deleteDoc`.
6. Keep the skeleton loaders — they now have a real purpose.

## Things that will bite
- **Storage CORS** blocks uploads from a GitHub Pages origin until configured. Expect this.
- **Firestore rules** default to deny-all in production mode. Write them early or every
  read fails silently and the app just looks empty.
- The app calls `history.pushState` for detail views and `history.back()` after saving a
  form. Async Firestore writes must resolve **before** the back navigation or the toast fires
  into a dead view.
- `refresh()` currently redraws everything synchronously. With snapshots, individual
  draw functions should be called from their own listeners instead.
- Don't let the Firebase config's `apiKey` scare anyone. It is a public identifier, not a
  secret, and ships in every Firebase web app. Security is entirely in the rules.

## Conventions in the existing code
- `$(id)` is `getElementById`. `esc()` escapes user content — use it on **everything**
  that comes from a user.
- `toast(msg)` for success, `toast(msg, 1)` for errors. No `alert()`.
- `pushDetail(title, html)` opens a full-screen view; `history.back()` closes it.
- Dates are ISO `YYYY-MM-DD` strings throughout, compared with `<` and `>=`.
  Don't introduce Date objects into stored data.
- CSS custom properties in `:root` hold the whole palette. Don't hardcode colours.

## Definition of done
A farmer signs up on a phone, photographs an animal, submits it, and sees "In review".
The office logs in, sees a badge on the Admin tab, approves it, and the animal appears in
Market for everyone — with the app installed to the home screen and opening fullscreen.
