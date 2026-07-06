# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A single-file interactive web app for supervisor conflict management training. Deployed at `https://supervisor-training.vercel.app` via Vercel (auto-deploys from `main`).

## Development

No build step. The entire app is `index.html` — open it directly in a browser or serve with any static file server:

```bash
python3 -m http.server 8080
# or
npx serve .
```

To deploy: push to `main`. Vercel picks it up automatically. `vercel.json` sets security headers (CSP, HSTS, nosniff, frame denial) — if you add a new external origin (CDN, API), update the CSP there or the browser will block it.

## Architecture

Everything lives in `index.html`:
- **CSS**: token-based design system in `:root` (`--ink`, `--indigo`, `--grad`, surface/text scales `--s*`/`--t*`, radii `--r-*`, shadows `--sh-*`), followed by component classes (topbar, cards, `.acc` accordions, `.opt` answer buttons, dashboard, quiz, print styles)
- **HTML**: static shell with empty containers (`#home-container`, `#skills-container`, `#scenarios-container`, `#styles-container`, `#playbook-container`, `#quiz-container`) filled by JS at runtime
- **JavaScript**: inline SVG icon registry (`ICONS`/`icon()`), auth, state, content data, and build functions

### Backend: Supabase

```
SUPABASE_URL / SUPABASE_ANON_KEY — hardcoded at top of <script> (anon key is publishable by design; RLS is the security boundary)
```

Single table: `user_progress` (RLS enabled — `auth.uid() = user_id` policy for all commands)
- `user_id` (FK to auth.users, unique)
- `scenarios_answered` (jsonb) — `{ [scenarioId]: { optIdx, correct } }`
- `quiz_best_score` (int)
- `quiz_attempts` (int)
- `skills_opened` (text[])

Supabase client is loaded from CDN via dynamic `import()` at boot. Row is created on first login, upserted on every interaction. Note: the free-tier project auto-pauses after inactivity — if production login fails with network errors, restore the project in the Supabase dashboard.

### App State

```js
userProgress = { scenarios_answered, quiz_best_score, quiz_attempts, skills_opened }
```

Loaded from Supabase in `onSignedIn()`, persisted via `saveProgress()` after every interaction. The home dashboard (`buildHome()`/`refreshHome()`) recomputes overall progress (`overallProgress()`: skills/7 + scenarios/6 + quiz-taken, averaged) after each interaction.

### Content Data

All training content is hardcoded in JS arrays — edit these to change what users see:

| Variable | Contents |
|---|---|
| `skillsData` | 7 conflict management skills (HTML content inline, uses component classes like `.compare`, `.step`, `.phrase`) |
| `scenarios` | 6 practice scenarios with options and feedback |
| `stylesData` | 5 conflict styles (Collaborating, Competing, etc.) |
| `questions` | 8 quiz questions with answer index |
| `PLAYBOOK` | 3 playbook phase cards |
| `MODULES` | home dashboard module cards (labels, per-module progress) |

If you change the number of skills/scenarios, update the counts in `overallProgress()`, `refreshHome()`, and `MODULES`.

### Section Flow

Nav pills are built by `buildNav()` from the `NAV` array; each maps to a `#sec-{name}` div toggled by `showSectionByName()`. `buildAll()` is called once after sign-in and populates all containers from the data arrays (it also restores saved scenario answers).

### Auth hardening (client side)

- `esc()` HTML-escapes user-controlled strings interpolated into `innerHTML` (currently the user's name on the dashboard) — keep using it for any new user-supplied content.
- Login lockout: 5 failed attempts / 15 min via `localStorage` (UX-level; Supabase enforces real server-side rate limits).
- Registration validates name ≤ 80, email format/length, password 6–72 chars.
