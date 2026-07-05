# Modern Premium Overhaul — Design Spec

**Date:** 2026-07-05
**Status:** Approved by Jeff
**Scope:** Complete visual + UX overhaul of the supervisor conflict-management training app (`index.html`).

## Goals

- Transform the app from "serviceable internal tool" to a polished, modern, premium-feeling learning product.
- Professional & serious tone — credible corporate training for supervisors at Uplift.
- Improve UX (dashboard home, per-module progress, quiz review) without changing features conceptually.

## Hard Constraints (must not change)

- Single-file `index.html`, no build step, deploys via push to `main` (Vercel).
- Same Supabase project, same auth flows (email/password login + register), same `user_progress` schema:
  `scenarios_answered` (jsonb), `quiz_best_score` (int), `quiz_attempts` (int), `skills_opened` (text[]).
  Existing users' saved progress must load and render correctly.
- All training content text preserved verbatim: 7 skills, 6 scenarios (including the uncommitted scenario #6 "Employee Arriving Late Repeatedly"), 5 styles, playbook, 8 quiz questions.
- Section names/tabs unchanged: home, skills, scenarios, styles, playbook, quiz.
- Print stylesheet preserved (updated for new classes).

## Design System

- **Typography:** Inter via Google Fonts (`system-ui` fallback stack). Tight modular type scale; large confident headings; 15–16px body.
- **Color:** deep navy anchor (~#0F1E33), single indigo→violet gradient accent, slate neutral scale. Green/red reserved exclusively for correct/incorrect semantics.
- **Surfaces:** soft layered shadows, 14–16px radii, card hover lift.
- **Motion:** 150–250ms eased transitions; `prefers-reduced-motion` respected.
- **Icons:** inline SVG line-style icon set with consistent stroke replaces all emoji in chrome (nav, headers, toasts, section titles). Emoji removed from UI chrome; content templates get proper component classes instead of inline styles.

## Structure

- **Sticky top bar:** brand mark, pill nav tabs (horizontally scrollable on mobile), initials avatar + sign out.
- **Auth screen:** redesigned to match brand (gradient panel, refined forms); login/register/error logic untouched.
- **Loading screen & toast:** rebranded to the new system.

## UX Upgrades

- **Home = dashboard:** time-of-day greeting with user's name; overall completion % computed as
  (skills_opened/7 + scenarios_answered/6 + (quiz taken ? 1 : 0)) / 3; progress bar; module cards with
  per-module progress ("4/6 completed", "Best 7/8") and continue affordance.
- **Skills:** accordion cards show a completed checkmark once opened; content re-laid-out with proper
  components (do/don't comparisons, numbered steps, phrase cards).
- **Scenarios:** answered-count in section header; smoother feedback reveal; saved answers restore as today.
- **Quiz:** improved progress bar and results screen; NEW — missed-question review at the end
  (question, your answer, correct answer). Score/attempt persistence logic unchanged.
- **Accessibility:** visible focus states, `aria-expanded` on accordions, semantic buttons, keyboard friendly.

## Verification

- Serve locally (`python3 -m http.server`), drive in a real browser via Playwright.
- Bypass auth locally (show `#app` + `buildAll()` via console injection) to inspect the signed-in experience.
- Check: every section, mobile viewport (375px), answered/unanswered scenario states, quiz full run
  including missed-question review, auth screen, print view sanity.
- Commit includes the pre-existing scenario #6 working-tree change.
