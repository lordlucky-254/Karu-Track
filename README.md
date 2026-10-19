# KARU TRACK — Cinematic Campus Discovery Network

A design prototype of a consent-first student discovery website for **Karatina University, Nyeri County**.
Everything is in a single file: **`index.html`** — no build step, no dependencies, no network calls.

> This is the **cinematic prototype** (simulated students, illustrative map). No GPS is read, nothing is stored or transmitted.

---

## 1. Open it

**Locally:** double-click `index.html`, or:

```bash
python3 -m http.server 8000
# then open http://localhost:8000
```

**In-app preview:** open `index.html` from the workspace — it renders fully because all art, icons, fonts and audio are inline.

---

## 2. What's built (mapped to the four scenes)

| Scene | Implemented |
|---|---|
| **1 · Cinematic opening** | Full-screen animated campus grid, drifting map lines, radar sweep, glowing logo, staggered title reveal, `Explore Campus` / `Get Started`, live demo stats, optional synthesized ambience. |
| **2 · Enter the live map** | Landing scales + fades into a full-screen pannable/zoomable campus map: boundary, river, A2 highway, buildings, sports ground, town, 16 student markers, 12 approved places, search, filters, live count. |
| **3 · Discover people** | Profile cards with verified badge, course/year, approximate or precise mode, distance, shared interests, an explanation of *why* you can see them, `Request contact`, `Locate`, `Block`, `Report`. Place cards add `Navigate` + `Set meeting point` with an animated demo route. |
| **4 · Control visibility** | Privacy panel with consent gate, sharing switch (off by default), audience (`Everyone / My faculty / Request only`), precision (`Approximate / Precise`), auto-stop timer (`15 min / 1 h / 4 h / Manual`), one-tap **Stop sharing now**, blocked accounts, reporting, withdraw-consent & clear data, Kenya DPA note. |

### Details worth clicking
- **Markers** pulse, scale inversely with zoom (never become huge blobs), and show a hover label.
- **Search** (`/` focusses it) matches names, courses, interests, areas and places.
- **Filters**: year chips + "Under 300 m".
- **Sharing on** draws your own accuracy radius on the map, starts a countdown pill, and auto-stops when it expires.
- **Request-only student** appears in the list with a hidden location and no map marker — demonstrating real privacy states, not just hidden buttons.
- **Block** removes the student from map + list and increments the blocked counter; **Clear** wipes consent, blocks, filters and sharing.
- **Reduce motion** toggle in *About* (also honours the OS `prefers-reduced-motion`).
- **Ambience** is synthesized with the Web Audio API (no audio files); speaker button = mute.

---

## 3. Design system

| Token | Value | Use |
|---|---|---|
| Midnight navy | `#050a14` → `#0a1526` | Backgrounds |
| Electric blue | `#4c9bff` | Primary actions, map geometry |
| Cyan | `#5ef0d4` | Live/consent states only |
| Amber | `#f0b96d` | Demo warnings, never for actions |
| Text | `#eaf2ff` / `#8ea6c9` | Primary / muted |
| Type | system sans, 10–11 px tracked caps for labels | Cinematic HUD feel |
| Motion | 0.25 s micro, 0.5–1.5 s camera & panels, cubic-bezier(.2,.85,.25,1) | Restrained, not busy |

Panels use dark glass (`backdrop-filter: blur(26px) saturate(135%)`), hairline borders, and a single soft glow per surface. Map markers are the only saturated elements so the eye always lands on people first.

---

## 4. Production workflow (phases 1–7)

### Phase 1 — Plan
- Confirm the **campus boundary** and approved places with the university (this prototype's geometry is illustrative).
- Define roles: `student`, `moderator`, `admin`.
- Write the location policy: what is stored, who can read it, how long it lives.

### Phase 2–3 — Frontend
```
karu-track/
├── index.html            # this prototype
└── src/                  # production port
    ├── main.tsx
    ├── routes/ Landing · Map · Privacy · Safety
    ├── components/ MapCanvas · StudentMarker · ProfileSheet · ConsentSheet
    ├── hooks/ useGeolocation · useSharingSession · useRealtimeStudents
    └── lib/ supabase.ts · geo.ts (buffer, snap-to-campus)
```
Stack: **React + Vite**, **Tailwind**, **Framer Motion**, **Leaflet + OpenStreetMap** tiles with a dark filter (`filter: invert(1) hue-rotate(180deg) brightness(.85) contrast(1.1)` or a CARTO dark basemap), `react-leaflet` markers replaced by custom `DivIcon`s to keep the glow/pulse treatment.

Port the three rules that already exist here:
1. Markers scale with zoom (`--inv` pattern) so the map stays legible.
2. Approximate mode renders a **buffered circle** (500 m), never the raw point.
3. Nothing renders for a student whose `sharing_state <> 'active'`.

### Phase 4 — Backend (Supabase)
```sql
-- profiles mirror auth.users
create table profiles (
  id uuid primary key references auth.users on delete cascade,
  display_name text not null,
  course text, year smallint check (year between 1 and 6),
  faculty text, interests text[] default '{}',
  verified boolean default false,
  created_at timestamptz default now()
);

-- one row per sharing session; history is intentionally short-lived
create table location_sessions (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references profiles(id) on delete cascade,
  mode text not null check (mode in ('precise','approx')),
  audience text not null check (audience in ('everyone','faculty','request')),
  geom geography(Point,4326) not null,   -- never exposed to clients
  accuracy_m int,
  started_at timestamptz default now(),
  expires_at timestamptz not null,
  ended_at timestamptz
);

-- what other students are allowed to read
create view visible_students with (security_invoker = true) as
select p.id, p.display_name, p.course, p.year, p.interests,
       case when s.mode = 'precise'
            then s.geom
            else st_snaptogrid(s.geom::geometry, 0.0045)::geography  -- ~500 m cell
       end as approx_geom,
       s.mode, s.audience
from profiles p
join location_sessions s on s.user_id = p.id
where s.ended_at is null and s.expires_at > now();
```
Row Level Security:
```sql
alter table location_sessions enable row level security;

create policy "own session read/write" on location_sessions
  for all using (auth.uid() = user_id) with check (auth.uid() = user_id);

-- coarse reads only, never raw geom
create policy "read active visible sessions" on location_sessions
  for select using (
    ended_at is null and expires_at > now()
    and audience = 'everyone' and auth.uid() <> user_id
    and not exists (select 1 from blocks b
                    where b.blocker_id = user_id and b.blocked_id = auth.uid())
    and not exists (select 1 from blocks b
                    where b.blocker_id = auth.uid() and b.blocked_id = user_id)
  );
```
- **Realtime** on `location_sessions` (or a thin `presence` table) — no polling.
- A scheduled job (`pg_cron`) that sets `ended_at = now()` for expired sessions and purges ended rows older than **24 h**.
- Contact requests in their own table with `pending / accepted / declined / expired` states; accept reveals nothing beyond the chosen precision.

### Phase 5 — Consent-based tracking
```
tap "share"  →  explain (mode, audience, expiry)  →  explicit confirm
             →  navigator.geolocation.watchPosition (only while a session is active)
             →  POST to edge function → validate:
                  · user verified
                  · session not expired
                  · coordinate inside campus buffer (or town area)
                  · accuracy sane (discard > 150 m fixes)
             →  write severity-reduced coordinate
stop / timer end / tab hidden > 30 min  →  session.ended_at = now()
```
Rules already encoded in the prototype and to keep in production:
- sharing **off by default**, consent is a separate, un-prechecked action;
- **one-tap stop**, visible at all times while sharing;
- **auto-expiry** is the default;
- **never** enable tracking on page load, and never silently re-enable after a reload;
- discard inaccurate fixes instead of broadcasting them.

### Phase 6 — Test before launch
- geolocation denied / granted / revoked mid-session; iOS Safari and Android Chrome;
- GPS drift and 150 m+ accuracy outliers; offline → reconnect;
- expired session disappears from other clients within seconds (Realtime);
- blocked user cannot read or request; RLS verified with a second account (not just UI);
- reported content reaches moderators; account deletion removes sessions;
- Lighthouse: first map paint, reduced motion, keyboard traps, contrast AA on dark glass.

### Phase 7 — Deploy
1. `npm run build` → deploy to **Vercel** (or Netlify), HTTPS enforced.
2. Environment variables: `VITE_SUPABASE_URL`, `VITE_SUPABASE_ANON_KEY` (service role key stays server-side only, in edge functions).
3. Supabase: run migrations, enable RLS on every table, set auth redirect URLs to the production domain.
4. Restrict CORS to the deployed origin; rate-limit session writes (e.g. 1 per 10 s per user).
5. Test on real phones over cellular, then a limited pilot with written consent.

---

## 5. Privacy & Kenya Data Protection Act, 2019

Before any public launch:
- [ ] Lawful basis documented: **explicit consent**, revocable, with a timestamped consent record.
- [ ] Data minimisation: no raw GPS history kept; ended sessions purged within 24 h.
- [ ] Purpose limitation: location used only to show presence to opted-in peers — never analytics or advertising.
- [ ] Transparency: short, plain-language notice (English + Kiswahili) covering what is collected, why, for how long, and who can see it.
- [ ] Data subject rights: access, correction, deletion, withdrawal — in-product, not by email only.
- [ ] Security: RLS on all tables, least-privilege keys, encryption in transit (HTTPS) and at rest.
- [ ] Minors/age policy, reporting + blocking, and a named data protection contact.
- [ ] **Register with the Office of the Data Protection Commissioner (ODPC)** and complete a data protection impact assessment for location processing before public deployment.

The prototype is a design artefact, not a compliance certificate.

---

## 6. Roadmap after the prototype

1. Port the UI to React/Vite and swap the vector map for Leaflet + OSM tiles.
2. Supabase auth + institutional verification (approved student list or university email if available — never assume every student has one).
3. Consent-based geolocation with the session model above.
4. Realtime presence, contact requests, moderators, and an audit log.
5. PWA packaging (manifest + service worker + offline shell) once HTTPS hosting is live.
