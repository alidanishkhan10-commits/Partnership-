# AUREA Partnership

A finance & business-analytics PWA for WD Textile. Reads live order data from the
existing **Factory Orders** app (one-way, read-only) and layers pricing, cost,
profit, and reporting on top — without ever modifying Factory Orders.

## Files

| File | Purpose |
|---|---|
| `index.html` | The entire app (HTML/CSS/JS in one file) |
| `manifest.json` | PWA manifest — name, icons, theme |
| `service-worker.js` | Offline caching (never caches live JSONBin calls) |
| `icon-192.png`, `icon-512.png`, `apple-touch-icon.png`, `favicon-32.png`, `favicon-16.png` | Generated from your uploaded AUREA logo |

## 1. Deploy to GitHub Pages

1. Create a new repo (e.g. `aurea-partnership`) and upload all files in this folder to the repo root.
2. Repo → **Settings → Pages** → Source: `main` branch, `/ (root)`.
3. Your app will be live at `https://<your-username>.github.io/<repo-name>/`.
4. Open it on your phone in Chrome → menu → **Install app** (or **Add to Home Screen**).

## 2. Required configuration (before it works fully)

Open `index.html`, find the `CONFIG` block near the top of the `<script>` tag:

```js
const FACTORY_BIN_ID  = "6a429e40da38895dfe112279";     // already filled in — same bin Factory Orders uses
const FACTORY_API_KEY = "$2a$10$yS5...";                 // already filled in

const AUREA_BIN_ID  = "";   // ← you need to fill this in
const AUREA_API_KEY = "";   // ← you need to fill this in

const ADMIN_PIN = "1234";   // ← change this before real use
```

**AUREA needs its own JSONBin bin** to store Partnership-only data (Selling Price,
Customer/Product overlay, and the hidden cost-calculation state). It must be a
**different bin** from Factory Orders — AUREA never writes to the Factory Orders bin.

1. Go to [jsonbin.io](https://jsonbin.io) and create a free account (or reuse the one
   you already used for Factory Orders).
2. Create a new bin with this starting content:
   ```json
   { "orders": {}, "windows": {} }
   ```
3. Copy the new **Bin ID** and your **X-Master-Key**, paste them into `AUREA_BIN_ID`
   and `AUREA_API_KEY`.
4. Change `ADMIN_PIN` to a PIN only you and your Admin know.
5. Commit and push — GitHub Pages redeploys automatically.

Until `AUREA_BIN_ID`/`AUREA_API_KEY` are filled in, the app still runs — it just
keeps Partnership data (pricing, customer/product info) in that device's local
storage only, and won't sync across your phone/laptop/Partner's phone.

## 3. How the data model works

Factory Orders' real schema (as read from its JSONBin) is:

```js
{ nonu: [...], wd: [ { id, name, qty, date, days, status, cuttingDate, dispatchedOn }, ... ] }
```

AUREA reads **only the `wd` array** (WD Textile) every 10 seconds, matches each
order by its permanent `id`, and merges in AUREA's own overlay record for that
`id`:

```js
{ customerName, productName, category, size, colour, sellingPrice,
  costPrice, profit, reductionPercent, pci, windowKey, priceBracket }
```

- `reductionPercent` and `pci` (Production Cost Index) are **hidden** — never
  rendered in any screen, table, or CSV export. Only `costPrice` (the result)
  ever reaches the UI.
- Selling Price is entered by **Admin only**, inside AUREA (Factory Orders has
  no price field at all). Cost Price is always auto-calculated from it — never
  editable directly.
- Production windows are 15-day blocks (1st–15th, 16th–end of month). The first
  order priced in a window+price-bracket combination generates a hidden base
  reduction %; later orders in the same window get a small persisted variation
  (±0.20%, occasionally up to ±0.30%). Editing Selling Price later reuses the
  same stored % — it never re-rolls. If an order's production start date
  (`cuttingDate`, falling back to order date) moves into a new window, a new %
  is generated for that window.

## 4. One important limitation — please read

The spec asks that the Reduction % and Production Cost Index **never be exposed
via DevTools, Console, or exports**. In a pure static, backend-free app (which
is what "runs entirely on GitHub Pages, no server" requires), that can only be
enforced at the *display* layer, not the *network* layer:

- The app never renders these values anywhere, and CSV/PDF exports never
  include them — that part is fully honored.
- But because there's no server to gate calculations, the raw AUREA JSONBin
  response (which does need to contain `reductionPercent`/`pci` so the app can
  recompute Cost Price offline) **is technically visible to anyone who opens
  the browser's Network tab** while using the app, and the formula itself is
  visible to anyone reading `index.html`'s source.
- True concealment would require a small server-side function (e.g. a
  Cloudflare Worker or Vercel serverless function) that computes Cost Price
  and returns only the final number — which means adding a minimal backend.
  That's a small, optional next step if you ever want airtight secrecy; happy
  to build it if you want it later.

For a small family business app, this is usually an acceptable trade-off, but
you should know about it rather than assume it's cryptographically hidden.

## 5. Two-firm note

Factory Orders tracks both **Nonu Garments** and **WD Textile** in one bin.
AUREA Partnership is scoped to **WD Textile only** (per your instruction) —
see `const FIRM = "wd";` near the top of the script. If you later want a Nonu
Garments partnership view too, this can be duplicated with `FIRM = "nonu"`
and its own AUREA bin.
