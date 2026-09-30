# Segovia Plans — project context for Claude Code

Map-based social app for IE University students in Segovia. Students see activities pinned to real places on a map of the city, join in one tap, and create their own. New activities appear on everyone's map instantly. The whole product is one loop: **create → appears on the map → others join**. Protect that loop's speed above everything else.

I'm a solo student developer with limited time and a zero budget. Prefer the simplest thing that works, free tiers only, and as few dependencies as possible. When in doubt, cut scope and tell me what you cut.

## Users and constraints
- Only IE students. Sign-up is restricted to emails ending in `@student.ie.edu`, enforced **server-side**, not just in the UI.
- Closed, trust-based community: real names, visible host on every activity, a report button. No anonymous anything.
- Mobile-first. Almost all use is on phones, often outdoors, one-handed. Design at 375px width first; desktop just needs to not break.
- Timezone is always Europe/Madrid for display. Store `timestamptz`.

## Stack (decided — don't swap without asking)
- Vite + React + TypeScript (strict) + Tailwind + shadcn/ui
- Supabase: auth, Postgres, realtime, storage (avatars). Free tier.
- Map: MapLibre GL via `react-map-gl/maplibre`, tiles from OpenFreeMap (`https://tiles.openfreemap.org/styles/liberty`, no API key). Keep the style URL in one constant so it can be swapped for MapTiler later.
- Data fetching: TanStack Query. Routing: React Router.
- Deploy: Vercel (static). Installable PWA (manifest + icons; no service-worker caching needed for MVP).
- Env vars: `VITE_SUPABASE_URL`, `VITE_SUPABASE_PUBLISHABLE_KEY` (the anon/publishable key) in `.env.local`. Never use or commit a service-role key in the frontend.

## MVP scope
In: auth + onboarding, map with pins, list view, create activity, activity detail, join/leave, host edit/cancel, My Activities, basic profile, report.

**Out (do not build yet, even if the Lovable reference has them):** chat/messages, waitlists, payments, venue/featured pins, push or email notifications, friends/following, ratings, recommendations. If something you're building would make one of these harder later, mention it, but don't build it.

## Auth
- Email OTP (6-digit code) via `signInWithOtp` + `verifyOtp`, not magic links and not passwords. Codes keep users in the same browser/PWA on mobile; magic links open the wrong browser.
- Server-side domain check: a Supabase "before user created" auth hook (Postgres function) that rejects non-`@student.ie.edu` emails. If the hook isn't available, fall back to a trigger. Also check in the UI for a friendly error.
- Every RLS policy additionally requires `public.is_ie_student()` (checks `auth.jwt()->>'email'` ends in `@student.ie.edu`).
- The Supabase built-in email sender has tiny rate limits. Custom SMTP (Resend free tier) must be configured before real users; remind me when we get there.
- After first login → onboarding (name required; program, year, avatar optional) → map.

## Data model
Write all schema changes as migrations in `supabase/migrations/`. Never edit the DB by hand.

- `profiles`: `id` (pk, = auth.users.id), `full_name` (2–60), `avatar_url` null, `program` null, `year` smallint null, `bio` (≤200) null, `created_at`.
- `activities`: `id` uuid, `host_id` → profiles, `title` (3–80), `description` (≤500) null, `category` enum (`sports`, `food`, `study`, `nightlife`, `culture`, `other`), `location_name` (≤80), `lat`, `lng` (check: within a generous Segovia bounding box, roughly lat 40.75–41.10, lng -4.35 to -3.85), `start_time` timestamptz, `max_participants` int null (null = unlimited; else 2–50), `participant_count` int not null default 0 (maintained by trigger), `status` enum (`active`, `cancelled`), `created_at`, `updated_at`.
- `participations`: `activity_id`, `user_id`, `created_at`, primary key `(activity_id, user_id)`. The host is inserted as a participant on create.
- `reports`: `id`, `reporter_id`, `target_type` (`activity` | `profile`), `target_id` uuid, `reason` (≤500), `created_at`. Insert-only for users; I review in the Supabase dashboard.
- Storage bucket `avatars`: public read, users write only to `avatars/{uid}/*`. Resize to ~256px client-side before upload.

### Rules (enforce in the database, not just the UI)
- Joining and leaving go through `join_activity(activity_id)` and `leave_activity(activity_id)` — `security definer` RPCs. Direct insert/delete on `participations` is blocked by RLS.
- `join_activity` locks the activity row (`select … for update`) and rejects: already joined, activity cancelled, activity already started, full (`participant_count >= max_participants`). Return a clear error code for each so the UI can show a specific message.
- The host can't leave their own activity; they cancel it instead.
- Host can edit title, description, category, location, time, and max. `max_participants` can't go below the current `participant_count`. Only the host can update; nobody deletes activities (cancel = `status = 'cancelled'`).
- `start_time` must be in the future and at most 60 days ahead on create.
- Map/list show `active` activities until `start_time + 1 hour` (labelled "Started" during that hour). After that they live only under Past in My Activities. Cancelled activities vanish from the map but show a "Cancelled" badge in participants' My Activities.
- RLS: all authenticated IE students can read profiles, activities, and participations. Users update only their own profile.

## Realtime
One channel subscribed to `postgres_changes` on `activities` (insert/update). Because `participant_count` lives on the activity row, this also keeps join counts live. On change, update the TanStack Query cache rather than refetching everything. Unsubscribe on unmount.

## Screens
Bottom tab bar: **Map · My Activities · Profile**. Floating "+" on the map.
1. **Map** — full-screen, centered 40.9481, -4.1184, zoom 14. Category-colored pins. "Locate me" button (browser geolocation, fail silently if denied). Filter chips: category + Today / This week / All. Map/List toggle (list sorted by soonest). Tapping a pin opens a bottom sheet: title, category, time, host, `3/4` count, Join button.
2. **Create activity** — pick location by tapping the map or choosing from a quick-pick list of known Segovia places (`src/data/places.ts`), then title, category, date, time, description, max (blank = no limit). Sensible defaults (e.g. today, next full hour). Must be completable in under 30 seconds.
3. **Activity detail** — everything above plus description, participants with avatars, "Directions" link (Google Maps URL with lat/lng), Join/Leave, host sees Edit/Cancel, Report in an overflow menu, share button (Web Share API, copy link fallback). Deep-linkable at `/a/:id`.
4. **My Activities** — tabs Hosting / Joined, each split into Upcoming and Past.
5. **Profile** — own profile with edit + logout; other users' profiles (read-only) with Report. Shows counts hosted/joined.

## Design
- Take visual direction from the Lovable reference (`lovable-reference/`) where it's good.
- One accent color per category, used identically on pins, badges, and chips. Define the category → color/icon/label map once in `src/lib/categories.ts`.
- Touch targets ≥ 44px, no hover-only interactions, bottom sheets over modals, respect safe-area insets, use `100dvh` for full-height layouts.
- Every screen needs loading, empty, and error states. The empty map state should push the user to create the first activity.

## Reference code
`lovable-reference/` is a clone of an earlier Lovable build of this app. Use it for UI ideas, layout, copy, and components worth porting. Don't copy its data model, auth, or scope blindly — this file wins where they disagree. It is gitignored; never modify it.

## How to work
- Work in the phases I give you. At the end of each phase: run `npm run typecheck`, `npm run lint`, and `npm run build`, then stop and tell me exactly what to test on my phone before we continue.
- Commit at the end of each phase with a clear message.
- For database changes, write the migration and tell me the command to apply it (`supabase db push`), unless the CLI is already linked and logged in, in which case run it.
- Don't add a dependency without saying why. Don't build abstractions for features that are out of scope.
- If something in this file seems wrong or will cause a problem, say so before building it.
