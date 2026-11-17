# Connecting the store to Supabase (shared with the Jageer Nepal app)

Once this is done, Mansa and Jageer Nepal use the **same Supabase project** — one database,
one URL, one pair of keys. Mansa's products, popup ad, uploaded images, admin login and orders all
live there. The site becomes plain files, so it can be hosted for free on GitHub Pages;
`server.py` is no longer needed.

Every table, function and storage bucket Mansa creates is named `mansa_...` (`mansa_products`,
`mansa_settings`, `mansa_orders`, `mansa_admins`, `mansa_is_admin()`, `mansa_save_store()`, the
`mansa_media` bucket). Nothing with that prefix exists in Jageer's project yet, so running the
steps below only adds new tables — it never reads, renames or drops anything Jageer already has.

Until `config.js` is filled in, the site keeps working on the local Python server exactly as before.

## 1. Use the existing Jageer Nepal project

No new project needed — open the same Supabase project Jageer Nepal already uses
(supabase.com → your organization → that project).

## 2. Create Mansa's tables

1. In that project, open **SQL Editor** → **New query**.
2. Paste the whole of [`supabase/schema.sql`](supabase/schema.sql) and click **Run**. It should say
   "Success". This only creates the `mansa_*` tables/functions/bucket above — it does not touch
   any of Jageer's own tables.
3. New query again: paste [`supabase/seed.sql`](supabase/seed.sql) and click **Run**. This loads
   Mansa's full catalogue (all products) and the popup ad into the `mansa_*` tables, replacing any
   `mansa_products` rows already there. If you import a new CSV later with
   `python3 tools/import_products.py file.csv`, it rewrites this file too; run it again in the SQL
   Editor.

## 3. Create (or reuse) your admin login

If you already have a login for Jageer's own admin/dashboard in this same project, you can reuse
that same account for Mansa — just do step 3 below with it. Otherwise create a new one:

1. **Authentication → Users → Add user → Create new user**: enter your email and a strong
   password, and tick **Auto Confirm User**.
2. In **SQL Editor**, run this with that account's email:
   ```sql
   insert into public.mansa_admins (user_id) select id from auth.users where email = 'you@example.com';
   ```
   Only accounts listed in `mansa_admins` can change the Mansa store — this is separate from
   whatever gates access to Jageer's own data, so adding someone here does not give them any
   Jageer permissions, and vice versa.

## 4. Connect the site

1. Open **Project Settings → API** (or the **Connect** button at the top) — the same URL and key
   Jageer's app already uses.
2. Copy the **Project URL** and the **anon / publishable** key.
3. Paste them into [`config.js`](config.js):
   ```js
   window.MANSA_SUPABASE = {
     url: "https://xxxxxxxx.supabase.co",
     anonKey: "eyJ... or sb_publishable_...",
   };
   ```
   Never use the **service_role** / **secret** key. `config.js` is public, and that key bypasses
   all the security rules (for Jageer's data too, since it's the same project).

## 5. Check it

1. Open the store. Products should load (now from Supabase).
2. Open `admin.html` and sign in with your **email and password**. You'll see a new **Orders** tab.
3. Place a test order on the store, then refresh **Orders** in the admin panel. It should appear
   there as well as on WhatsApp.
4. In the Supabase **Table Editor**, confirm Jageer's own tables are untouched and unchanged.

## 6. Host it on GitHub Pages (optional)

1. Push the code to GitHub.
2. In the repository, open **Settings → Pages**, set **Source** to "Deploy from a branch", choose
   `main` and `/ (root)`, then **Save**.
3. After a minute the site is live at `https://<your-username>.github.io/<repo-name>/`, and the
   admin panel at `.../admin.html`.

## Good to know

- **Images**: new uploads go to the Supabase `mansa_media` storage bucket (separate from any
  bucket Jageer's app uses). The built-in images in `images/` stay in the site files and are
  listed in `images/manifest.json`. If you add a file to `images/` by hand, add it to that list
  too.
- **Isolation**: because every Mansa object is prefixed `mansa_`, you can drop/rebuild Mansa's
  tables at any time (re-run `schema.sql`) without any risk to Jageer's data, and the two apps'
  Row Level Security policies and `is_admin`-style functions don't interact.
- **Orders** store the prices the customer saw at checkout. Always confirm the final price on
  WhatsApp.
- **Free plan**: Supabase pauses free projects that get no traffic for about a week. Normal store
  visits (from either app) count as activity.
- **Going back to local mode**: empty the two values in `config.js`. The local mode uses
  `store.json` and `ad.json`, which won't include changes made while connected to Supabase.
