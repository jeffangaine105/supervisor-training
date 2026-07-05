# Modern Premium Overhaul Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rebuild the look, feel, and UX of the single-file supervisor training app to a modern premium standard while preserving all content, auth, and progress-persistence behavior.

**Architecture:** Everything stays in `index.html` (CSS + HTML shell + JS). The CSS is rewritten as a token-based design system; emoji are replaced by an inline-SVG icon registry in JS; the home section becomes a computed dashboard; the quiz gains a missed-question review. Supabase auth/persistence code is untouched except where render functions change output markup.

**Tech Stack:** Vanilla HTML/CSS/JS, Inter via Google Fonts, Supabase JS v2 via CDN (unchanged), Playwright MCP for browser verification, `python3 -m http.server` for local serving.

## Global Constraints

- Single file `index.html`; no build step; no new dependencies beyond the Google Fonts stylesheet link.
- Supabase config, auth functions (`doLogin`, `doRegister`, `doLogout`, `onSignedIn`, `onSignedOut`, `initSupabase`), and `saveProgress()` upsert shape must remain byte-for-byte in behavior (markup-only edits allowed inside them, e.g. `user-name-chip` → avatar).
- `user_progress` schema unchanged: `scenarios_answered` jsonb, `quiz_best_score` int, `quiz_attempts` int, `skills_opened` text[].
- All content strings in `skillsData`, `scenarios` (6 entries incl. #6 "Employee Arriving Late Repeatedly"), `stylesData`, `questions`, and playbook text preserved verbatim. HTML *around* the text may change; the human-readable words may not.
- Section ids stay `sec-home|skills|scenarios|styles|playbook|quiz`; container ids stay `skills-container`, `scenarios-container`, `styles-container`, `playbook-container`, `quiz-container`, plus new `home-container`.
- No emoji in UI chrome (nav, headers, buttons, toasts). Emoji inside preserved content strings are allowed where they are part of content (e.g. ❌/✅ compare labels may be replaced by styled text labels "Avoid"/"Say this" since those are decoration, not content).
- Professional & serious tone; green/red used only for correct/incorrect semantics.
- `prefers-reduced-motion` respected; visible `:focus-visible` states; `aria-expanded` on accordions.
- Print stylesheet preserved and adapted to new class names.
- Verification for every task = browser check (no JS test framework in this project; adding one is out of scope).

## Verification Harness (used by every task)

Start server once: `python3 -m http.server 8080` (background, from repo root).

To inspect the signed-in app without real auth, run in the browser console / `browser_evaluate`:

```js
// AUTH BYPASS (local verification only — never committed)
document.getElementById('loading-screen').style.display='none';
document.getElementById('auth-screen').style.display='none';
document.getElementById('app').style.display='block';
userProgress = { scenarios_answered:{1:{optIdx:1,correct:true},2:{optIdx:0,correct:false}}, quiz_best_score:7, quiz_attempts:2, skills_opened:['s1','s2','s3'] };
currentUser = { id:'test', email:'jane@test.com', user_metadata:{ full_name:'Jane Smith' } };
window.__testMode = true;   // saveProgress() guard is currentUser-null check; set a fake user but expect network errors to be swallowed
buildAll(); renderProgressStrip();
```

Note: `saveProgress()` will fire real Supabase requests with a fake user id; they fail harmlessly (RLS) and are awaited without error handling — wrap verification interactions expecting console noise, which is acceptable locally.

---

### Task 1: Design system CSS + HTML shell + icon registry

**Files:**
- Modify: `index.html` — `<head>` (fonts), full `<style>` block, auth screen markup, `#app` header/nav markup, section headers, toast/loading markup.

**Interfaces:**
- Produces CSS custom properties consumed by all later tasks:
  `--ink --navy --indigo --violet --grad --green --green-bg --green-line --red --red-bg --red-line --amber --cyan --s0 --s1 --s2 --t1 --t2 --t3 --t4 --line --r-sm --r-md --r-lg --sh-sm --sh-md --sh-lg --ease`
- Produces JS: `const ICONS = {...}` (inline SVG strings, 20×20 viewBox, `stroke="currentColor" stroke-width="1.8" fill="none"`) with keys:
  `logo, home, skills, scenarios, styles, playbook, quiz, check, chevron, user, logout, reset, arrowRight, target, sparkle, alert, book`
  and helper `function icon(name, cls='')` returning `<span class="icon ${cls}">${ICONS[name]}</span>`.
- Produces component classes consumed later: `.card`, `.acc` (accordion), `.acc-head`, `.acc-body`, `.pill-nav`, `.nav-pill`, `.btn`, `.btn-primary`, `.btn-ghost`, `.btn-sm`, `.badge`, `.sec-head`, `.sec-title`, `.sec-sub`.

- [ ] **Step 1: Add fonts + rewrite `<style>`**

`<head>` gains:

```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Inter:opsz,wght@14..32,400..800&display=swap" rel="stylesheet">
```

Replace the whole `<style>` block with the new design system. Token block (exact values):

```css
:root{
  --ink:#0B1526;--navy:#12233D;--indigo:#4F46E5;--violet:#7C3AED;
  --grad:linear-gradient(135deg,#4F46E5 0%,#7C3AED 100%);
  --green:#059669;--green-bg:#ECFDF5;--green-line:#A7F3D0;
  --red:#DC2626;--red-bg:#FEF2F2;--red-line:#FECACA;
  --amber:#B45309;--cyan:#0E7490;
  --s0:#FFFFFF;--s1:#F7F8FB;--s2:#EEF1F6;
  --t1:#0B1526;--t2:#3D4A5E;--t3:#64748B;--t4:#94A3B8;
  --line:#E4E8EF;
  --r-sm:10px;--r-md:14px;--r-lg:20px;
  --sh-sm:0 1px 2px rgba(11,21,38,.05),0 1px 8px rgba(11,21,38,.04);
  --sh-md:0 2px 6px rgba(11,21,38,.06),0 8px 24px rgba(11,21,38,.07);
  --sh-lg:0 4px 12px rgba(11,21,38,.10),0 24px 56px rgba(11,21,38,.14);
  --ease:cubic-bezier(.4,0,.2,1);
}
body{font-family:'Inter',system-ui,-apple-system,'Segoe UI',sans-serif;background:var(--s1);color:var(--t2);-webkit-font-smoothing:antialiased}
@media (prefers-reduced-motion:reduce){*,*::before,*::after{animation-duration:.01ms!important;transition-duration:.01ms!important}}
:focus-visible{outline:2px solid var(--indigo);outline-offset:2px}
.icon{display:inline-flex;flex-shrink:0}.icon svg{width:20px;height:20px}
```

Include full component CSS for: sticky topbar (`.topbar` — dark navy, brand mark w/ gradient logo tile, pill nav that horizontally scrolls on mobile via `overflow-x:auto`), avatar chip (initials in gradient circle), auth screen (full-viewport gradient `radial-gradient` navy backdrop, white card `--sh-lg`, refined inputs), cards, accordions (`.acc[data-open]` + animated chevron), buttons, badges, toast (bottom pill, `--sh-lg`), loading screen (navy, pulsing gradient mark), print block.

- [ ] **Step 2: Rewrite shell markup**

Auth screen: brand mark (SVG logo tile) replaces 🧑‍💼; tabs/forms keep existing ids (`l-email`, `l-pass`, `r-name`, `r-email`, `r-pass`, `btn-login`, `btn-register`, `auth-error`, `auth-success`, `login-form`, `register-form`) so auth JS is untouched.

Header becomes:

```html
<header class="topbar">
  <div class="topbar-inner">
    <div class="brand"><span class="brand-mark" id="brand-mark"></span>
      <div><div class="brand-name">Supervisor Training</div><div class="brand-sub">Conflict Management</div></div></div>
    <nav class="pill-nav" id="nav"></nav>
    <div class="topbar-user"><span class="avatar" id="user-avatar"></span><button class="btn-ghost btn-signout" onclick="doLogout()">Sign out</button></div>
  </div>
</header>
```

Nav is now built by JS (Task 1 Step 3) from a `NAV` array so icons render; the old `.progress-strip` div is removed (dashboard replaces it). Home section body becomes `<div id="home-container"></div>`. Section headers get `icon()` injected by JS or static inline SVG — use static markup with a `.sec-head` flex row.

- [ ] **Step 3: Add `ICONS`, `icon()`, `buildNav()`, and updated `showSection`**

```js
const NAV = [
  { id:'home',      label:'Home',      icon:'home' },
  { id:'skills',    label:'Skills',    icon:'skills' },
  { id:'scenarios', label:'Scenarios', icon:'scenarios' },
  { id:'styles',    label:'Styles',    icon:'styles' },
  { id:'playbook',  label:'Playbook',  icon:'playbook' },
  { id:'quiz',      label:'Quiz',      icon:'quiz' },
];
function buildNav(){
  document.getElementById('nav').innerHTML = NAV.map(n =>
    `<button class="nav-pill${n.id==='home'?' active':''}" data-target="${n.id}" onclick="showSectionByName('${n.id}')">${icon(n.icon)}<span>${n.label}</span></button>`).join('');
}
function showSectionByName(id){
  document.querySelectorAll('.section').forEach(s=>s.classList.toggle('active', s.id==='sec-'+id));
  document.querySelectorAll('.nav-pill').forEach(b=>b.classList.toggle('active', b.dataset.target===id));
  window.scrollTo({top:0,behavior:'smooth'});
}
```

Delete the old `showSection(id, btn)` (textContent-matching variant of `showSectionByName` is now canonical; all callers use `showSectionByName`). `onSignedIn` sets `#user-avatar` initials (`name.split(/\s+/).map(w=>w[0]).slice(0,2).join('').toUpperCase()`) with `title` = full name, and calls `buildNav()` before `buildAll()`.

- [ ] **Step 4: Verify in browser**

Serve, open `http://localhost:8080`, confirm: auth screen shows new brand + Inter font, no emoji; run auth bypass snippet; topbar renders with 6 pill tabs + avatar "JS"; tab switching works; no console errors other than expected Supabase RLS noise.

- [ ] **Step 5: Commit**

```bash
git add index.html && git commit -m "Overhaul: design system, topbar, auth screen, icon registry"
```

---

### Task 2: Dashboard home

**Files:**
- Modify: `index.html` — add `buildHome()`, dashboard CSS; remove `renderProgressStrip()` and all 4 call sites (replace with `refreshHome()`).

**Interfaces:**
- Consumes: `icon()`, tokens, `userProgress`, `currentUser`, `showSectionByName`.
- Produces: `function overallProgress()` → int 0–100; `function refreshHome()` (rebuilds `#home-container`; safe to call anytime after sign-in); module meta array `MODULES`.

- [ ] **Step 1: Implement progress math + dashboard renderer**

```js
function overallProgress(){
  const sk = Math.min(userProgress.skills_opened.length,7)/7;
  const sc = Math.min(Object.keys(userProgress.scenarios_answered).length,6)/6;
  const qz = userProgress.quiz_best_score!==null?1:0;
  return Math.round((sk+sc+qz)/3*100);
}
function greeting(){
  const h = new Date().getHours();
  return h<12?'Good morning':h<17?'Good afternoon':'Good evening';
}
```

`buildHome()` renders into `#home-container`:
1. Hero card: `${greeting()}, ${firstName}` (firstName = full_name first word or email prefix), completion % with animated progress bar (`.meter` filled by `--grad`), and three stat chips (Skills x/7 · Scenarios x/6 · Quiz best or "Not attempted").
2. "Why Conflict Management Matters" card — existing copy verbatim, restyled (indigo left rule, checklist with SVG check icons).
3. Module grid (5 cards from `MODULES = [{id:'skills',...},{id:'scenarios',...},{id:'styles',...},{id:'playbook',...},{id:'quiz',...}]`), each card: icon tile, title, per-module progress line (`"3/7 reviewed"`, `"2/6 answered"`, `"Best 7/8 · 2 attempts"`, styles/playbook show static blurbs), arrowRight affordance, `onclick="showSectionByName(id)"`.
4. Quote card: existing quote verbatim on navy panel.

`refreshHome(){ if(currentUser) buildHome(); }` — called from `buildAll()`, `answerScenario`, `resetScenario`, `trackSkill`, `nextQuestion` (replacing every `renderProgressStrip()` call).

- [ ] **Step 2: Verify in browser**

With bypass state (3 skills, 2 scenarios, quiz 7): hero shows 47% (3/7+2/6+1 = .4286+.3333+1 = 1.762/3 = 58.7 → **59%** — verify computed number matches the formula, not this arithmetic, in-browser), stat chips correct, module cards navigate. Also verify zeroed progress renders "Not attempted".

- [ ] **Step 3: Commit**

```bash
git add index.html && git commit -m "Overhaul: dashboard home with computed progress"
```

---

### Task 3: Skills section

**Files:**
- Modify: `index.html` — skills CSS components, `skillsData` content templates, `buildSkills()`, `toggleCard()`.

**Interfaces:**
- Consumes: `.acc` classes, `icon()`, `trackSkill(id)` (unchanged persistence).
- Produces: component classes reused by Task 4/5: `.compare` (avoid/say-this pair), `.step` (numbered step row), `.phrase` (italic phrase card), `.gap-grid`/`.gap-cell` (diagnose grid).

- [ ] **Step 1: Restyle accordion + content components**

`buildSkills()` output per skill: `.acc` card with number tile (replaces emoji icon; skill number 01–07 in gradient-tinted tile using each skill's `color`), title/sub, completed check (`icon('check','done')` shown when `userProgress.skills_opened.includes(s.id)`), chevron. Header button gets `aria-expanded` synced in `toggleCard`:

```js
function toggleCard(id, skillId){
  const acc = document.getElementById(id+'-acc');
  const open = acc.toggleAttribute('data-open');
  acc.querySelector('.acc-head').setAttribute('aria-expanded', open);
  if (open && skillId) trackSkill(skillId);
}
```

Rewrite each `skillsData[i].content` replacing inline styles with the new components; **every human-readable sentence stays verbatim**. ❌/✅ glyphs in compare labels become styled text labels ("Avoid" / "Say this") via `.compare` CSS. `trackSkill` additionally flips the check icon on the open card without full rebuild.

- [ ] **Step 2: Verify in browser**

All 7 skills expand/collapse; opened skills show check; `aria-expanded` toggles (inspect via snapshot); content text matches original wording; compare/step/phrase components render cleanly at 375px width.

- [ ] **Step 3: Commit**

```bash
git add index.html && git commit -m "Overhaul: skills accordion and content components"
```

---

### Task 4: Scenarios, Styles, Playbook

**Files:**
- Modify: `index.html` — `buildScenarios()`, `buildStyles()`, `buildPlaybook()`, `answerScenario()`, `resetScenario()`, related CSS.

**Interfaces:**
- Consumes: `.card`, `.acc`, `.compare`, `.gap-grid` from Task 3; `saveProgress()` unchanged.
- Produces: scenario DOM ids unchanged (`sc-{id}`, `sc-{id}-opt-{i}`, `sc-fb-{id}`, `sc-reset-{id}`) so `buildAll()` answer-restore keeps working; section-header count `#scen-count`.

- [ ] **Step 1: Scenarios**

Section header gains `<span class="sec-count" id="scen-count"></span>` updated by `updateScenCount()` (`"4 of 6 answered"`), called from `buildAll`/`answerScenario`/`resetScenario`. Option buttons: letter chip, refined hover, feedback panel slides open (CSS `grid-template-rows` transition). Correct/incorrect visuals keep green/red semantics; toast copy: `'Correct — progress saved'` / `'Saved — review the feedback'` (no emoji). Reset button uses `icon('reset')`.

- [ ] **Step 2: Styles + Playbook**

Styles: same accordion pattern as skills, monogram tile per style with its `color`, tag badge, when/how two-pane body (verbatim text). Key-principle panel restyled navy. Playbook: three phase cards (Before/During/After) with icon + numbered `.pb-row`s, then the diagnose `.gap-grid` (shared component with skill 6). All text verbatim.

- [ ] **Step 3: Verify in browser**

Answer a scenario → state persists visually, count updates, feedback animates; reset works; restored answers (from bypass seed) render answered state on load; styles/playbook render correctly at 375px and desktop.

- [ ] **Step 4: Commit**

```bash
git add index.html && git commit -m "Overhaul: scenarios, styles, playbook"
```

---

### Task 5: Quiz + missed-question review

**Files:**
- Modify: `index.html` — `quiz` state object, `renderQuiz()`, `answerQuiz()`, `nextQuestion()`, `resetQuiz()`, quiz CSS.

**Interfaces:**
- Consumes: `questions`, `userProgress.quiz_best_score/quiz_attempts`, `refreshHome()`.
- Produces: `quiz.responses` array of chosen indices (parallel to `questions`), used only by results renderer.

- [ ] **Step 1: Track responses + results review**

```js
let quiz = { current:0, score:0, answered:false, done:false, responses:[] };
function answerQuiz(idx){
  if (quiz.answered) return;
  quiz.answered = true;
  quiz.responses[quiz.current] = idx;
  /* existing correct/incorrect marking, restyled classes */
}
function resetQuiz(){ quiz={current:0,score:0,answered:false,done:false,responses:[]}; renderQuiz(); }
```

Results screen: score dial (large number + label, gradient ring via `conic-gradient`), best-score/attempts line, message tiering unchanged (>=7 / >=5 / else, same copy), then **"Review missed questions"** list — for each `i` where `quiz.responses[i] !== questions[i].answer`: question text, "Your answer: …" (red tint), "Correct answer: …" (green tint). If none missed, show a "Perfect score — nothing to review" card. Buttons: Retake (primary), Review Skills (ghost).

- [ ] **Step 2: In-quiz polish**

Progress bar uses `--grad`; question counter + running score; option buttons match scenario option styling; next button appears after answering (unchanged flow).

- [ ] **Step 3: Verify in browser**

Full 8-question run with 2 deliberate misses → results show 6/8, exactly those 2 in review with correct your/correct answers; perfect run shows the perfect-score card; retake resets; best score/attempts update on the dashboard.

- [ ] **Step 4: Commit**

```bash
git add index.html && git commit -m "Overhaul: quiz flow and missed-question review"
```

---

### Task 6: Polish, print, mobile, full verification

**Files:**
- Modify: `index.html` — print block, toast/loading final pass, meta description/og tags (keep same copy; bump `og:image` untouched), any dead CSS/JS removal (`renderProgressStrip`, old `.progress-strip`, `showSection`, unused classes).

- [ ] **Step 1: Dead code sweep + print styles**

Print block updated to new class names: hide `.topbar`, auth, loading, buttons; force `.section` and `.acc-body` visible; white background; borders instead of shadows.

- [ ] **Step 2: Full browser verification pass**

Desktop (1280) and mobile (375): auth screen, all 6 sections, accordion states, answered/unanswered scenarios, complete quiz run incl. review, toast, focus-visible via keyboard Tab, reduced-motion spot check (emulate), no console errors (except bypass RLS noise), print preview sanity via `media: print` emulation.

- [ ] **Step 3: Final commit + push**

```bash
git add index.html && git commit -m "Overhaul: polish, print styles, dead code removal"
git push origin main   # only after user-visible confirmation step per session norms — push deploys to production
```

---

## Self-Review Notes

- Spec coverage: design system (T1), dashboard+% formula (T2), skills checkmarks (T3), scenarios count/restore + styles + playbook (T4), quiz review (T5), a11y/print/motion (T1/T3/T6), verification (harness + T6). Auth/schema/content constraints in Global Constraints. ✔
- Scenario #6 already in working tree; committed with Task 1's first commit implicitly (whole-file commits). ✔
- Type consistency: `icon(name, cls)`, `showSectionByName(id)`, `refreshHome()`, `overallProgress()`, `quiz.responses` used consistently across tasks. ✔
