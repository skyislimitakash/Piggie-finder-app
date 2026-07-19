# Pigie Finder — Build Plan

A community platform where college students discover, review, and directly contact trusted local vendors and service providers near their campus.

**Tech stack:** Node.js, Express, EJS, CSS, JavaScript (vanilla), SQL (MySQL/PostgreSQL), Google Maps API

**Revision note:** This version fixes a data-model gap (review photos), adds missing constraints/indexes, addresses production-readiness items (session store, CSRF, error handling), and expands every phase into concrete, ordered sub-steps.

---

## 1. Architecture Overview

Pigie Finder follows a classic **server-rendered MVC architecture**, since EJS is a server-side templating engine rather than a client-side framework. This keeps the stack simple, SEO-friendly, and easy to reason about for a solo/small-team build.

```
┌─────────────────────────────────────────────────────────────┐
│                        CLIENT (Browser)                       │
│   EJS-rendered HTML + CSS + vanilla JS + Google Maps JS SDK   │
└───────────────────────────┬────────────────────────────────┘
                             │ HTTP(S)
┌───────────────────────────▼────────────────────────────────┐
│                     EXPRESS APPLICATION                       │
│  ┌───────────┐  ┌────────────┐  ┌────────────┐  ┌─────────┐ │
│  │  Routes   │→ │ Controllers│→ │  Services/ │→ │  Models │ │
│  │ (URLs)    │  │  (logic)   │  │  (Maps API,│  │  (SQL)  │ │
│  │           │  │            │  │  distance) │  │         │ │
│  └───────────┘  └────────────┘  └────────────┘  └────┬────┘ │
│  Middleware: auth, session, multer, csrf, validation, errors  │
└───────────────────────────┬──────────────────────────┬──────┘
                             │                          │
                  ┌──────────▼─────────┐     ┌──────────▼──────────┐
                  │   SQL Database      │     │  External Services   │
                  │  (MySQL/Postgres)   │     │  Google Maps API      │
                  │  users, vendors,    │     │  (Geocoding, Places,  │
                  │  reviews, colleges, │     │   Maps JS SDK)        │
                  │  images, categories │     │  Image storage (local │
                  │                     │     │  disk or Cloudinary)  │
                  └─────────────────────┘     └───────────────────────┘
```

### How the tech stack maps to the architecture

| Layer | Tech | Responsibility |
|---|---|---|
| View | EJS + CSS + vanilla JS | Renders pages server-side; JS handles map rendering, form interactivity, star ratings, image previews |
| Application | Node.js + Express | Routing, request handling, middleware (auth, sessions, file upload, validation, CSRF, errors) |
| Services | Plain JS modules | Google Maps geocoding calls, distance calculations — kept out of controllers so they're testable/reusable |
| Data | SQL (MySQL/PostgreSQL) | Persistent storage for users, colleges, vendors, reviews, images, categories |
| Integration | Google Maps API | Vendor location pins, navigation/directions, geocoding + Places Autocomplete, distance-based search |
| Auth | express-session (DB-backed store) + bcrypt | Login/session management, password hashing |
| File handling | multer | Vendor/review image uploads |

---

## 2. Database Schema (SQL)

Changes from the first draft are marked with comments.

```sql
-- Colleges act as the "community hub" students belong to
CREATE TABLE colleges (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(150) NOT NULL,
  address VARCHAR(255),
  latitude DECIMAL(10,7),
  longitude DECIMAL(10,7)
);

CREATE TABLE users (
  id INT PRIMARY KEY AUTO_INCREMENT,
  college_id INT,
  name VARCHAR(100) NOT NULL,
  email VARCHAR(150) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  club_name VARCHAR(100),         -- e.g., "Sports Club" (nullable)
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (college_id) REFERENCES colleges(id),
  INDEX idx_users_college (college_id)          -- NEW: speeds up "vendors near my college" + club lookups
);

CREATE TABLE categories (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(80) NOT NULL UNIQUE          -- NEW: UNIQUE prevents duplicate "Food"/"food" entries
);

CREATE TABLE vendors (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(150) NOT NULL,
  category_id INT,
  college_id INT,               -- which college's "nearby area" this belongs to
  address VARCHAR(255),
  latitude DECIMAL(10,7),
  longitude DECIMAL(10,7),
  contact_number VARCHAR(20),
  whatsapp_number VARCHAR(20),
  avg_cost DECIMAL(8,2),        -- CHANGED: numeric cost instead of free-text price_range, so it's filterable/sortable
  added_by INT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (category_id) REFERENCES categories(id),
  FOREIGN KEY (college_id) REFERENCES colleges(id),
  FOREIGN KEY (added_by) REFERENCES users(id),
  INDEX idx_vendors_college_category (college_id, category_id),   -- NEW: core query path (Phase 4/8)
  INDEX idx_vendors_location (latitude, longitude)                 -- NEW: for distance filtering
);

CREATE TABLE reviews (
  id INT PRIMARY KEY AUTO_INCREMENT,
  vendor_id INT NOT NULL,
  user_id INT NOT NULL,
  rating TINYINT NOT NULL CHECK (rating BETWEEN 1 AND 5),
  comment TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,  -- NEW: supports editable reviews
  FOREIGN KEY (vendor_id) REFERENCES vendors(id),
  FOREIGN KEY (user_id) REFERENCES users(id),
  UNIQUE KEY uq_review_per_user_vendor (vendor_id, user_id)   -- NEW: one review per user per vendor (editable, not stackable)
);

-- CHANGED: images now optionally attach to a specific review (a visit),
-- not only to the vendor in general — matches the original spec:
-- "reviews and pictures for the services they have used"
CREATE TABLE vendor_images (
  id INT PRIMARY KEY AUTO_INCREMENT,
  vendor_id INT NOT NULL,
  review_id INT,                -- NEW: nullable — NULL = general vendor photo, set = tied to a specific review
  image_url VARCHAR(255) NOT NULL,
  uploaded_by INT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (vendor_id) REFERENCES vendors(id),
  FOREIGN KEY (review_id) REFERENCES reviews(id) ON DELETE CASCADE,
  FOREIGN KEY (uploaded_by) REFERENCES users(id),
  INDEX idx_images_vendor (vendor_id)
);

CREATE TABLE favorites (
  user_id INT,
  vendor_id INT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,   -- NEW: lets you sort "recently saved"
  PRIMARY KEY (user_id, vendor_id),
  FOREIGN KEY (user_id) REFERENCES users(id),
  FOREIGN KEY (vendor_id) REFERENCES vendors(id)
);
```

**Design decisions worth confirming with your own use case before you build:**
- One review per user per vendor (editable) was chosen to keep average ratings meaningful. If you'd rather let students log every separate visit, drop the `UNIQUE` constraint and instead show "N visits, avg rating" on vendor pages.
- `avg_cost` is a single number rather than a range — simplest for MVP filtering ("under ₹200"). You can upgrade to `min_cost`/`max_cost` later without breaking anything else.

---

## 3. Folder Structure

```
pigie-finder/
├── build.md
├── .env
├── .env.example              # NEW: committed template so teammates know which vars to set
├── .gitignore
├── package.json
├── server.js
├── config/
│   ├── db.js                 # SQL connection pool
│   └── session.js            # NEW: DB-backed session store config
├── routes/
│   ├── authRoutes.js
│   ├── vendorRoutes.js
│   ├── reviewRoutes.js
│   ├── collegeRoutes.js
│   ├── searchRoutes.js
│   └── favoriteRoutes.js      # NEW
├── controllers/
│   ├── authController.js
│   ├── vendorController.js
│   ├── reviewController.js
│   ├── searchController.js
│   └── favoriteController.js  # NEW
├── models/
│   ├── userModel.js
│   ├── vendorModel.js
│   ├── reviewModel.js
│   ├── collegeModel.js
│   └── imageModel.js          # NEW
├── services/                  # NEW: matches architecture diagram — Maps API + distance logic
│   ├── geocodeService.js
│   └── distanceService.js
├── middleware/
│   ├── authMiddleware.js
│   ├── uploadMiddleware.js    # multer config
│   ├── validateMiddleware.js
│   ├── csrfMiddleware.js      # NEW
│   ├── rateLimitMiddleware.js # NEW: throttles auth routes
│   └── errorMiddleware.js     # NEW: centralized 404 / 500 handling
├── views/
│   ├── partials/
│   │   ├── header.ejs
│   │   ├── footer.ejs
│   │   └── navbar.ejs
│   ├── auth/
│   │   ├── login.ejs
│   │   └── register.ejs
│   ├── vendors/
│   │   ├── list.ejs
│   │   ├── detail.ejs
│   │   └── addEdit.ejs
│   ├── errors/
│   │   ├── 404.ejs            # NEW
│   │   └── 500.ejs            # NEW
│   ├── index.ejs
│   └── profile.ejs
├── public/
│   ├── css/
│   │   └── style.css
│   ├── js/
│   │   ├── map.js             # Google Maps rendering logic
│   │   ├── upload-preview.js
│   │   └── rating.js
│   └── uploads/                # locally stored vendor/review images (dev only — see Phase 11)
└── database/
    ├── schema.sql
    └── seed.sql
```

---

## 4. Build Phases

### Phase 0 — Planning & Requirements
1. Write down the final feature list and freeze scope for MVP: vendor discovery, reviews with photos, direct contact info, maps, club-based filtering. Explicitly list what's **out of scope** for v1 (e.g., admin dashboard, notifications) so it doesn't creep into earlier phases.
2. Define user roles: **Student** (default, can browse/add vendors/review) and an optional **club tag** on the profile (not a separate role — just a label used for filtering, per Phase 9).
3. Decide the review model now (see schema note above): one editable review per user per vendor.
4. Sketch low-fidelity wireframes for: Home, Vendor List (with map), Vendor Detail, Add/Edit Vendor, Login/Register, Profile. Doesn't need to be polished — just enough to know what data each page needs.
5. Set up the GitHub repo, add `.gitignore` (node_modules, .env, public/uploads/*), and create one issue/checklist item per phase below.
6. Register for a Google Cloud project and enable: Maps JavaScript API, Geocoding API, Places API. Generate an API key but don't restrict it yet (do that in Phase 6 once you know your domain).

### Phase 1 — Environment Setup
1. `npm init -y`, then set `"type": "commonjs"` (default) and add scripts to `package.json`:
   ```json
   "scripts": {
     "start": "node server.js",
     "dev": "nodemon server.js",
     "db:migrate": "node database/migrate.js",
     "db:seed": "node database/seed.js"
   }
   ```
2. Install core dependencies:
   `npm i express ejs mysql2 dotenv bcrypt express-session express-mysql-session multer connect-flash csurf express-rate-limit express-validator`
3. Install dev dependency: `npm i -D nodemon`
4. Create `.env.example` (committed) and `.env` (gitignored) with:
   ```
   PORT=3000
   DB_HOST=localhost
   DB_USER=root
   DB_PASSWORD=
   DB_NAME=pigie_finder
   SESSION_SECRET=change_me
   GOOGLE_MAPS_API_KEY=your_key_here
   NODE_ENV=development
   ```
5. Build `server.js`: initialize Express, set `view engine` to `ejs`, serve `public/` as static, mount `express.urlencoded`/`express.json`, wire up session middleware (placeholder store for now), and a catch-all 404 handler at the bottom.
6. Add a temporary `GET /` route rendering `views/index.ejs` with "Pigie Finder — coming soon" to confirm the pipeline works end-to-end before building real features.
7. Run `npm run dev` and verify hot-reload works on file changes.

### Phase 2 — Database Design & Setup
1. Install MySQL locally, or provision a free-tier cloud instance (PlanetScale, Railway, or Supabase for Postgres — adjust `AUTO_INCREMENT`/`SERIAL` syntax if you go Postgres).
2. Create the database: `CREATE DATABASE pigie_finder;`
3. Save the full schema from Section 2 into `database/schema.sql` and run it against your DB.
4. Write `database/seed.sql` with: 2–3 sample colleges, 6–8 categories (Food, Printing, Stationery, Laundry, Sports Gear, Repairs, Salon, Pharmacy), and 3–5 sample vendors per college so you have data to build UI against immediately.
5. Build `config/db.js` as a connection pool (not single connections) using `mysql2/promise`:
   ```js
   const mysql = require('mysql2/promise');
   module.exports = mysql.createPool({
     host: process.env.DB_HOST, user: process.env.DB_USER,
     password: process.env.DB_PASSWORD, database: process.env.DB_NAME,
     waitForConnections: true, connectionLimit: 10
   });
   ```
6. Build `config/session.js` using `express-mysql-session` pointed at the same pool, so sessions survive server restarts and work if you ever scale to multiple instances.
7. Add a temporary `/health` route that runs `SELECT 1` through the pool to confirm connectivity, then delete it once confirmed (or keep it for deployment health checks).

### Phase 3 — Authentication System
1. `userModel.js`: `createUser()`, `findByEmail()`, `findById()` — all parameterized queries, never string-concatenated SQL.
2. `POST /auth/register`: validate input with `express-validator` (name, valid email, password min length, college selected) → hash password with `bcrypt.hash(pw, 12)` → insert user → auto-login by creating the session.
3. `POST /auth/login`: look up by email → `bcrypt.compare()` → on success, regenerate the session (`req.session.regenerate`) and store `userId`/`collegeId` → on failure, flash a generic "invalid email or password" (don't reveal which field was wrong).
4. `authMiddleware.js`: a `requireAuth` function that checks `req.session.userId`, redirects to `/auth/login` with a flash message if absent. Apply it to any route that adds/edits vendors, posts reviews, or manages favorites.
5. `POST /auth/logout`: `req.session.destroy()`, clear the session cookie, redirect home.
6. Apply `rateLimitMiddleware` to `/auth/login` and `/auth/register` (e.g., 10 requests/15 min per IP) to blunt brute-force attempts.
7. Build `login.ejs` / `register.ejs` with `connect-flash` error/success messages and a CSRF hidden field (wire up `csurf` here since it's the first form in the app).

### Phase 4 — Vendor Core (CRUD)
1. `vendorModel.js`: `create()`, `findById()`, `findByCollege(collegeId, filters)`, `update()`, `delete()` (soft-delete via a `deleted_at` column is worth considering here instead of hard delete, so reviews/images aren't orphaned).
2. `GET /vendors`: fetch vendors for the logged-in user's college (join `categories` for display name, join a rating subquery for avg rating) → render `list.ejs`. Add pagination (`LIMIT`/`OFFSET`, 12–20 per page) now rather than retrofitting it later.
3. `GET /vendors/:id`: fetch vendor + its category + average rating + review count + all reviews (with reviewer name, rating, comment, and any attached images) + general vendor images → render `detail.ejs`.
4. `GET /vendors/new` and `POST /vendors`: form with name, category dropdown (populated from `categories` table), address (wired to Places Autocomplete in Phase 6), contact number, WhatsApp number, avg cost. Server-side validation middleware rejects missing name/category/contact before hitting the DB.
5. `GET /vendors/:id/edit` and `PUT /vendors/:id`: only allow if `req.session.userId === vendor.added_by` (ownership check) — this is easy to forget and is a real authorization gap if skipped.
6. Confirm all vendor queries filter by `college_id` server-side (not just in the UI) so one college's data never leaks into another's list via a crafted request.

### Phase 5 — Reviews & Image Uploads
1. `reviewModel.js`: `upsertReview()` (insert or update, respecting the `UNIQUE(vendor_id, user_id)` constraint — use `ON DUPLICATE KEY UPDATE` in MySQL), `findByVendor()`, `getAverageRating(vendorId)`.
2. Review form on the vendor detail page: star rating (1–5, built as a small `rating.js` widget — radio inputs styled as stars, works without JS too) + comment textarea + optional photo upload input (multiple files allowed).
3. `uploadMiddleware.js` (multer): restrict to image MIME types, cap file size (e.g., 5MB), cap file count per submission (e.g., 4), and generate randomized filenames (don't trust the original filename) to `public/uploads/`.
4. On review submit: save the review row, then for each uploaded file insert a row into `vendor_images` with `review_id` set (per the schema fix above). A "general vendor photo" upload (not tied to a review) simply leaves `review_id` NULL.
5. Display: vendor detail page shows a photo grid combining general + review-attached images, and each review shows its own attached photos inline.
6. Reminder for `detail.ejs`: render `comment` with `<%= %>` (auto-escaped), never `<%- %>`, since it's untrusted user input.

### Phase 6 — Google Maps Integration
1. Restrict the API key (HTTP referrer restriction to your dev + prod domains) now that Phase 1 registered it.
2. `services/geocodeService.js`: wraps a call to the Geocoding API — given an address string, returns `{ lat, lng }`. Called server-side from the vendor create/edit controller so lat/lng are always stored alongside the address.
3. Add **Places Autocomplete** to the address `<input>` on the Add/Edit Vendor form (`public/js/map.js` or a dedicated `places-autocomplete.js`) — this gives cleaner addresses and reduces geocoding failures compared to free-typed text.
4. `public/js/map.js`, vendor list page: render a map with a pin per vendor currently shown in the list (pass vendor lat/lng/name from EJS into a JSON array the script consumes) — clicking a pin scrolls to / highlights the matching list card.
5. Vendor detail page: single pin + a "Get Directions" link/button that opens Google Maps directions (`https://www.google.com/maps/dir/?api=1&destination=lat,lng`) in a new tab — no API call needed for this, just a URL scheme.
6. Handle the failure case: if geocoding fails (bad/incomplete address), don't silently store NULL lat/lng — flash a message asking the user to refine the address, or store the college's fallback coordinates.

### Phase 7 — Frontend Polish (EJS + CSS + JS)
1. Build `header.ejs` (meta tags, CSS/JS links, CSRF meta tag for AJAX calls if any), `navbar.ejs` (logo, search bar, login/profile links), `footer.ejs`.
2. `style.css`: mobile-first (students primarily browse on phones) — define spacing/color variables at the top, then base styles, then component styles (cards, buttons, forms, star ratings), then a `@media (min-width: 768px)` block for desktop layout.
3. Client-side JS: `upload-preview.js` (show thumbnails before submitting), `rating.js` (interactive star selection), and basic form validation feedback (required-field highlighting) as a UX layer on top of — not a replacement for — server-side validation.
4. Empty/loading states: "No vendors added near your college yet — be the first!" on empty vendor lists, a skeleton/spinner while the map loads, and a friendly message on failed geocoding.
5. `errorMiddleware.js` + `views/errors/404.ejs` / `500.ejs`: catch unmatched routes and thrown errors centrally instead of leaking Express's default stack-trace pages, especially once `NODE_ENV=production`.

### Phase 8 — Search & Filtering
1. `GET /vendors/search`: accepts query params — `category`, `maxCost`, `minRating`, `keyword`, `sort` (rating / distance / newest).
2. `searchController.js`: build the SQL query with parameter placeholders assembled conditionally (never interpolate raw query params into the SQL string) — e.g. build a `WHERE` clause array and a matching params array, then join them.
3. Search bar in the navbar (keyword) + a filter sidebar/drawer on the vendor list page (category checkboxes, cost slider, min-rating selector).
4. Distance sort: compute distance via the Haversine formula, either as a raw SQL expression (fast, scales) or in JS after fetching a bounded set (simpler to write, fine at MVP scale) — pass the student's current location (via `navigator.geolocation`, with a manual fallback if they decline) into the query.
5. Preserve filters in the URL query string so results are shareable/bookmarkable and survive a page refresh.

### Phase 9 — Community/Club Features
1. Add a "Club" field to the profile edit form so students can self-tag (e.g., "Sports Club", "Coding Club", or blank).
2. `GET /vendors/:id` and `GET /vendors?club=Sports+Club`: add a query path that filters/highlights vendors added or most reviewed by members of a given club — directly supports the AIT sports-club example from the spec.
3. Vendor detail page: show a "Recommended by N Sports Club members" badge when applicable (computed via a join between `reviews`/`vendors.added_by` and `users.club_name`).
4. `favoriteController.js` + `favoriteRoutes.js`: add/remove favorite (toggle button on vendor cards), and a "My Favorites" view on the profile page.

### Phase 10 — Testing
1. Manual end-to-end checklist: register → login → add vendor (with geocoded address) → upload vendor photo → submit a review with photos → edit that review → search/filter → view on map → get directions → favorite/unfavorite → logout.
2. Cross-account test: create two users in two different colleges, confirm neither sees the other's vendors in list/search, and confirm a user can't edit a vendor they didn't add (authorization check from Phase 4).
3. Input-hardening test: submit a review comment containing `<script>` tags and confirm it renders as literal text, not executable script (validates the `<%= %>` escaping decision from Phase 5).
4. SQL injection spot-check: try `' OR '1'='1` in the search/login fields and confirm parameterized queries reject it cleanly.
5. Responsive check across at least: a small phone width (~375px), tablet (~768px), and desktop (~1280px).
6. Optional but recommended before scaling the team: add `jest` + `supertest` for a handful of route-level smoke tests (register, login, create vendor) so future changes don't silently break auth.

### Phase 11 — Deployment
1. Choose hosting: Render/Railway for the Node app + PlanetScale/Railway MySQL (or Supabase for Postgres), or a single VPS if you want full control.
2. Move all `.env` values into the hosting provider's environment variable settings — never commit real secrets, and rotate the Google Maps key if it was ever pushed to a public repo.
3. Set `NODE_ENV=production`; confirm `errorMiddleware.js` suppresses stack traces in that mode.
4. Since `public/uploads/` on most PaaS platforms is ephemeral (wiped on redeploy), migrate image storage to a persistent option before going live — either a mounted volume (VPS) or a cloud bucket (Cloudinary/S3), swapping the storage target in `uploadMiddleware.js` only.
5. Point the DB-backed session store at the production database, and confirm cookies are set `secure: true` + `sameSite: 'lax'` once served over HTTPS.
6. Run `database/schema.sql` against the production DB, then seed real (not test) categories and colleges.
7. Set up a custom domain (optional) and HTTPS (usually automatic on Render/Railway), then re-restrict the Google Maps API key to the final production domain.

### Phase 12 — Future Enhancements (post-MVP)
- Admin dashboard to moderate vendors/reviews/reports (flagging system).
- Notifications when a favorited vendor gets a new review.
- Vendor verification badge (college-confirmed vendors).
- REST/JSON API layer if a mobile app is planned later (routes already separated from views, so this is a controller-reuse job, not a rewrite).
- Multi-college support with a college-switcher for students who transfer or visit another campus.
- Soft-delete + audit trail for vendors/reviews instead of hard deletes, if moderation becomes necessary.

---

## 5. Suggested Build Order Summary

`Phase 0 → 1 → 2 → 3 → 4 → 5 → 6 → 7 → 8 → 9 → 10 → 11`

Phases 0–3 form the foundation (auth + DB), Phases 4–6 deliver the core value proposition (vendor discovery + maps), Phases 7–9 add polish and the community differentiator, and Phases 10–11 get it live.