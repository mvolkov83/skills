---
name: playwright-web-qa
description: Hands-on manual / exploratory QA of web applications driven through the Playwright MCP browser tools (`browser_navigate`, `browser_snapshot`, `browser_find`, `browser_click`, `browser_type`, `browser_fill_form`, `browser_console_messages`, `browser_network_requests`, `browser_resize`, `browser_take_screenshot`, `browser_evaluate`, `browser_run_code_unsafe`) — the way a world-class human tester works, not a click-through of the happy path. Grounded in Session-Based Test Management, the Heuristic Test Strategy Model and FEW HICCUPPS oracles, Hendrickson's Test Heuristics Cheat Sheet, Whittaker's tours, ISTQB test design, OWASP WSTG, WCAG 2.2 AA, Nielsen Norman Group heuristics and severity ratings, Baymard form research, Core Web Vitals, and BBST bug advocacy. Covers the test charter (mission, top risks, budget), oracles and the session log, environment safety, Playwright MCP mechanics (snapshot-first interaction, `browser_find`, waiting on state, console with `all: true` and network after every step, slow typing for key handlers, HttpOnly cookies, Service-Worker-blind routing, CDP throttling, isolated profiles and per-role storage state, device emulation vs resize), user flows from object life histories (happy / alternate / error / out-of-order / invalid state transitions / abandon-and-resume / first-time / per-role, conflicting edits in two tabs, one-time actions and replay, following data into search, exports and emails), per-screen functional heuristics (zero / one / many, none / some / all, list sort-filter-search-pagination, three-value boundary analysis, number and date attacks, paste and autofill, bypassing client-side validation, Back after logout and bfcache, server-side logout, session expiry, network failure, breakpoints, outputs and print), accessibility against WCAG 2.2 AA (axe-core scan per state, names / roles / states, keyboard and focus-not-obscured, forms and announced errors, status messages, contrast without rounding, 320 px reflow, 200 % zoom, text spacing, forced colors), UX evaluation (all ten Nielsen heuristics as concrete checks, cognitive-walkthrough questions per step, NN/g response-time limits, error-message and confirmation-dialog guidelines, measured LCP / CLS / interaction latency, Nielsen 0–4 severity with an evidence standard), measured visual and layout quality (shared edges, spacing scale and proximity, Baymard form composition — single column, field width, labels not placeholders, required and optional marking — type-scale inventory, line length and height, truncation, layout shift, sticky layering, image sharpness, design-token conformance), risk-picked exploratory tours, **product opportunities** (missing-feature gaps such as data shown but not filterable, no bulk action, no undo, lost filter state, no export, dead-end navigation — each grounded in evidence, valued, capped at five, never implemented without a decision), **Boy Scout mode** (widen the net, fix reproduced defects at the root with a failing-then-passing test, keep tidyings apart from behaviour fixes, confirmation then regression, bounded loop), and reporting (RIMGEA bug reports with severity separate from priority, the three-part testing story, coverage matrix, explicit untested areas). Use this skill whenever the user asks to test, QA, check, verify, smoke-test, click through, explore, audit accessibility of, review the UX or layout of, or "see if it works" for a web page, frontend, SPA, admin panel, landing page, checkout, sign-up / login / onboarding flow, or a just-implemented UI change using Playwright MCP or "the browser". Trigger even when the request is small ("open localhost:3000 and check the form", "does the new modal work?", "is the layout of this form OK?", "check alignment", "is this accessible?") or the user only says "what's missing?", "what would improve this?", "boy scout", "polish it", "make it shine", "go through the user flows", or "check the UX". Do NOT use for writing automated test code with `@playwright/test`, Cypress, Selenium or pytest-playwright (that is test authoring, not browser-driven QA), for load / performance benchmarking, for non-web targets (native mobile, desktop apps, CLI), or for analysing a user story without a running app (use `manual-qa-toolkit:analyse-story` for that).
---

# Playwright Web QA — exploratory testing, user flows, UX, layout, Boy Scout mode

This skill turns the Playwright MCP browser tools into a professional tester's workbench. The
tools provide *hands* (navigate, click, type, snapshot); this skill provides the *judgment*: what
to look at, how to decide something is a problem, how to walk user flows the way real users do,
how to evaluate the experience and the layout rather than just the function, and how to report —
or, in Boy Scout mode, fix — what was found.

The default failure mode of browser-driven testing is **happy-path confirmation**: open the page,
fill the form with perfect data, see a success toast, declare "works". That catches almost nothing.
Real defects live in the second attempt, the empty state, the skipped step, the second tab, the
slow network, the Back button, the 320 px viewport, the error message nobody read, and the flow
next to the one being tested. The second failure mode, specific to an AI tester, is **invented
expectations**: reporting a "bug" whose expected behaviour was never grounded in anything. Rule
2 exists for that.

Testing here is exploratory in the canonical sense: learning, test design, execution and
evaluation run in parallel, steered by what each result reveals — not a script followed to the
end and then a bit of "exploration" at the finish.

Scope: the Playwright MCP tool set (`mcp__playwright__browser_*`). For requirement analysis and
formal test-case design before a session, use `manual-qa-toolkit` (`analyse-story`,
`generate-test-cases`, `assess-risk`). For regression tests of backend defects, defer to
`pytest-best-practices`. Canonical sources behind each section are listed in
`references/sources.md`.

## 1. Charter, oracles, and safety

1. **Write a charter before the first navigation.** One mission line — *Explore <target>
   with <resources, techniques, data> to discover <information: risks, quality criteria>* — plus:
   environment and URL, build / commit, change under test, in-scope flows, personas / roles and
   the credentials for each, test data, mode (scoped or Boy Scout, §9), a time or flow budget, and
   the **top 3–5 risks** ("what would hurt most if this broke?" — money, auth, data loss, legal
   first). Derive flows and risks from AC, the PR diff, or the ticket when they exist. Ask only for
   what is genuinely missing (credentials, environment URL); infer the rest.
2. **Name the oracle for every problem.** "Expected" must come from something checkable
   (FEW HICCUPPS): an explicit **claim** (AC, spec, help text, UI copy, marketing), consistency
   with the **product** itself (the same thing elsewhere in the app), its **history** (previous
   behaviour), its **image**, **comparable** products, **user** purpose, the **world** (arithmetic,
   dates, physics), **statutes and standards** (WCAG, legal copy), or behaviour the tester cannot
   **explain**. When the only oracle is a guess, log the item as an **issue** (a question for the
   user), not a bug.
   - **Why:** an agent can confidently invent an expectation. A bug with no oracle is an opinion,
     it wastes triage time, and a few of them destroy trust in the whole report.
3. **Keep a session log from the first action**, as separate lists: **notes** (what was
   covered, with which data, what was learned), **bugs**, and **issues** (open questions,
   blockers, testability problems), and **opportunities** (missing features, §8). Mark each entry *on-charter* or *opportunity*. Write it to a
   scratch file on long sessions so context compaction does not lose it. The report (§10) is built
   from this log, not from memory.
4. **Confirm the environment from evidence before any state-changing action** — hostname,
   environment banner, API base URL in network requests — never by assumption. Local / dev /
   staging: act freely. Production, and any environment wired to live third parties (real payment
   keys, real SMTP / SMS, outbound webhooks): read-only for those actions unless the user
   explicitly authorises the specific action. On shared environments avoid disrupting others:
   account lockouts, rate-limit exhaustion, bulk deletes, global settings.
   - **Why:** a QA session submits forms dozens of times with edge-case data. Against live
     integrations that is dozens of real charges, orders, or messages — none of them undoable.
5. Use clearly synthetic test data (`qa+<scenario>@example.com`, names prefixed `QA`), never
   real people's personal data, so artefacts created by the session are identifiable and
   cleanable. Record everything created in the log.
6. **Bring the app up yourself** when the codebase is available — do not ask for a URL. Learn
   how the project runs from its own sources (a project launch skill, `CLAUDE.md`, README, task
   runners such as `Makefile` / `package.json` scripts, compose files, `.env.example`), reuse a dev
   server that already serves the working tree, otherwise start dependencies, migrations, seed data,
   and the app as documented, and wait for readiness on evidence (health endpoint, "listening" log
   line), never a fixed sleep. Ask only for what the project cannot provide (a missing secret, VPN,
   or undocumented start command). Stop what you started at the end (rule 106); leave what
   was already running.

## 2. Playwright MCP mechanics

7. **No Playwright MCP, no session.** Before the charter's first action, confirm the
   `browser_*` tools are available and respond. If they are missing, disconnected, or failing
   (server not configured, "failed to connect", every call erroring), stop and ask the user to
   restore the Playwright MCP server (reconnect via `/mcp`, fix its config, or restart the client)
   — then continue once it is back. Never substitute a poorer instrument: no hand-written
   Playwright / Puppeteer / Selenium scripts, no `curl` or HTTP fetches of the page, no other
   browser-automation tool, no reading the source to "infer" how the UI behaves. Apply the same
   rule if the tools drop mid-session: pause, report what was covered so far, ask for the restore.
   - **Why:** a substitute tool silently changes what is being tested — no accessibility tree, no
     console and network capture, no real interaction — so its "pass" is not evidence, while the
     report still reads as if the browser session happened.
8. **Snapshot first, act second.** `browser_snapshot` returns the accessibility tree
   with element refs; interact via those refs (tools also accept a unique selector as `target`).
   Use `browser_find` (text or regex) to locate an element cheaply instead of a full snapshot on
   large pages, and `browser_snapshot` with `target` / `depth` to scope it. Use
   `browser_take_screenshot` for visual checks and as evidence, not for navigation.
   - **Why:** screenshots cost many tokens and cannot be clicked; the snapshot is cheaper and is
     what actions are addressed against.
9. **Refs go stale after DOM changes.** Actions return an updated snapshot — act on the
   latest one. After changes the session did not trigger (timers, pushes, background refresh),
   take a fresh snapshot before acting.
10. **Wait on state, never on time.** `browser_wait_for` takes `text` / `textGone`. For
    state without text (a spinner, `aria-busy`), use `browser_run_code_unsafe`:
    `async page => page.locator('[aria-busy="true"]').first().waitFor({ state: 'detached' })`.
    A fixed `time` wait either wastes time or races the app.
11. **After every meaningful action, check both side channels.**
    `browser_console_messages` with `level: "warning"` and `all: true` (errors, unhandled
    rejections, framework warnings, CSP violations, failed resource loads), and
    `browser_network_requests` with `static: false` and a `filter` regex for the API path (4xx /
    5xx, duplicate requests, requests on every keystroke, oversized payloads, slow calls). Drill in
    with `browser_network_request` (`part: "response-body"` etc.). A UI that looks fine while the
    console throws is a defect.
    - **Why:** most SPA failures are silent in the UI — a swallowed 500, a retry storm, a state
      update on an unmounted component. And the console tool defaults to messages *since the last
      navigation*: without `all: true`, errors logged just before a redirect or full-page submit
      are silently lost.
12. Use `browser_fill_form` for speed, but exercise keystroke behaviour (search-as-you-type,
    input masks, character counters, on-input validation) with `browser_type` and `slowly: true`,
    plus `browser_press_key` for Tab, Enter, Escape.
    - **Why:** without `slowly`, `browser_type` fills the whole value at once and per-key handlers
      never run — the exact behaviour under test is skipped.
13. `browser_handle_dialog` (`accept`, `promptText`) for native `alert` / `confirm` /
    `prompt` / `beforeunload`; browsers show `beforeunload` only after a real interaction with the
    page. An unexpected native dialog is itself a finding (usually poor UX).
14. `browser_resize` changes the viewport only — no device pixel ratio, touch, or mobile
    user agent. For a real mobile check the MCP server must run with `--device "<name>"`; say in
    the report which was used. `browser_emulate_media` covers `colorScheme`, `reducedMotion`,
    `forcedColors`, `contrast`, and `media: "print"`. `browser_tabs` for multi-tab scenarios,
    `browser_navigate_back` for history.
15. `browser_snapshot` with `boxes: true` gives element geometry cheaply;
    `browser_evaluate` gives computed styles, `scrollWidth`, storage contents, and anything
    measurable in the DOM. Read cookies with `browser_run_code_unsafe`
    (`async page => page.context().cookies()`).
    - **Why:** `document.cookie` cannot see HttpOnly cookies — which is where session cookies
      live — so session and security checks built on it are wrong.
16. **Fault injection** via `browser_run_code_unsafe` and the Playwright `page` API:
    `page.context().setOffline(true)`, `page.route('**/api/**', r => r.abort())`,
    `r.fulfill({ status: 500 })`, delayed `r.continue()`. First check
    `navigator.serviceWorker.controller`: requests served by a Service Worker (PWAs, MSW) bypass
    `page.route`, so injection silently does nothing. Clean up with
    `page.unrouteAll({ behavior: 'ignoreErrors' })`.
17. **Slow network and CPU** (Chromium only); reset both before continuing (latency 0,
    throughput -1, rate 1):

     ```js
     async page => { const s = await page.context().newCDPSession(page);
       await s.send('Network.emulateNetworkConditions',
         { offline: false, latency: 400, downloadThroughput: 50000, uploadThroughput: 20000 });
       await s.send('Emulation.setCPUThrottlingRate', { rate: 4 }); }
     ```
18. **The Playwright MCP profile is persistent by default** (cookies, storage, login
    state, HTTP cache survive sessions). For first-time-user runs and per-role logins prefer an
    `--isolated` server with `--storage-state=<role>.json`. Otherwise clear cookies
    (`context.clearCookies()`), `localStorage`, `sessionStorage`, IndexedDB and `caches`, and
    unregister service workers. A rebuilt asset may come from the HTTP cache — disable it via CDP
    `Network.setCacheDisabled` or use a cache-busting query before concluding "the fix didn't work".
19. A persistent profile can be used by one browser at a time. "Browser is already in use"
    means another instance holds the lock: `browser_close` it, or — only if it is a stale instance
    this session started — kill it and remove the profile's `Singleton*` lock files. Parallel
    agents need `--isolated` or distinct `--user-data-dir`s.
20. Geolocation and permissions (`context.grantPermissions`, `context.setGeolocation`),
    iframes (`page.frameLocator(...)`), downloads (`page.waitForEvent('download')`) and popups
    (`page.waitForEvent('popup')`) go through `browser_run_code_unsafe`. `browser_file_upload`
    takes absolute paths inside the workspace roots.

## 3. User flows — discovery and walking

User flows are the backbone of the session. Walking complete flows as a user with a goal finds far
more than inspecting screens in isolation, because defects cluster at the seams between steps.

21. **Build a flow inventory first.** Sources: AC / ticket / PR, the app's navigation and
    primary CTAs, routes in the codebase, role-specific areas — and **object life histories**: for
    each core entity, create → view → edit → state changes → delete / archive, including deleting
    something other records reference. Write each flow as
    `Persona → goal → entry point → steps → success state`; mark the in-scope ones. A missing
    update or delete path, or an orphaned dependent, is a finding.
22. **Walk each flow as a persona with a goal, not a tester with a script.** At every
    step ask the four cognitive-walkthrough questions: (1) will the user try to achieve the right
    result? (2) will they notice the correct action is available? (3) will they associate that
    action with their goal? (4) after acting, will they see progress toward the goal? Any "no"
    fails the step — record which question failed; it is a UX finding (§6) even when nothing is
    broken.
23. **Walk every in-scope flow in these variants** (skip only what cannot apply):
    - **Happy path** — realistic, not perfect, data.
    - **Alternate paths** — every branch the UI offers (other payment method, skip optional step,
      edit from the summary step).
    - **Error paths** — invalid input at each step, server error at each submit (rule 16).
    - **Out of order** — deep-link to a later step without completing the earlier ones, repeat a
      completed step, resubmit a step from history. The server must refuse or redirect, never
      accept a skipped prerequisite (confirmation without payment).
    - **Every state transition** — for status-driven entities (draft / submitted / approved /
      cancelled), each valid transition and, one per attempt, each invalid one (edit after submit,
      cancel after ship).
    - **Abandon and resume** — leave mid-flow (close tab, navigate away, reload), come back: is
      progress kept, discarded cleanly, or corrupted? A leave-page warning where data would be lost,
      and none where nothing would be.
    - **First-time vs returning user** — empty states, onboarding, defaults vs pre-filled data.
    - **Per role** — the same flow as each role; a forbidden role is blocked in the UI *and* at the
      direct URL.
24. **Verify the outcome, not the toast.** After a flow reports success, confirm the effect
    where the user would look: the item appears in its list with correct values, counters and
    badges update, the detail page matches input, and the change survives reload and re-login.
    Then **follow the data out**: it is findable in search, correct in filtered lists, reports and
    exports, and any email or notification shows the same values (check the test inbox if the
    charter provides one). A success message over an unsaved change is a critical defect.
25. **Cross-flow consistency.** State produced by flow A is reflected correctly in flow B
    (edit profile → name updated in header, order list, emails; delete item → gone from search,
    dashboards, recent lists).
26. **Repeat the flow immediately.** A second run catches stale state, unreset forms,
    duplicate-key errors, cached wrong data, and "works only once" bugs.
27. **Conflicting edits.** Open the same record in two tabs (or as two users) and save
    different changes from each: the second save must be detected (conflict warning, reload
    prompt), not silently overwrite. Delete in one tab, then edit in the other.
    - **Why:** lost updates never show in single-tab testing and corrupt data silently.
28. **Limits and replay.** Actions meant to happen once (apply coupon, redeem invite, claim
    bonus, vote, confirm payment) fail cleanly the second time — including via Back + resubmit and
    from a second tab.
29. Record per-flow status in the coverage matrix (rule 104): variant × result ×
    findings.

## 4. Functional heuristics — every screen

Run this pass on every screen a flow touches. It is the minimum, not the ceiling.

**States and collections**
30. Each data view has four designed states — loading, empty, error, populated — plus
    partial (some widgets failed). Force each one (fresh account, route abort, slow response): no
    blank area, no eternal spinner, no raw error text, no layout jump.
31. **Count, selection, position.** For every list: 0, 1, and many items (and the maximum the
    UI allows); for bulk actions, none / some / all selected, including "select all" across pages;
    act on the first, a middle, and the last item — and the last item of the last page. Delete the
    last row of a page and check the empty-page handling and the counters.
32. **List operations.** Sort each sortable column both ways (ties, empty values, accented and
    mixed-case text, numbers stored as text); combine filter + search + sort + pagination; the state
    survives reload, Back, and a shared URL; totals and page counts match the filtered set.
33. Long and short content: long names, unbroken strings, many decimals, one-character values,
    100+ items. Truncation, wrapping, overflow, pagination / virtualisation — and truncated text is
    reachable in full (rule 82).

**Forms and input**
34. **Per field, three-value boundaries**: min−1, min, min+1, max−1, max, max+1 (length and
    value). Plus empty, whitespace-only, leading / trailing spaces, wrong type, Unicode and emoji,
    RTL text, newlines, and a markup string (`<b>x</b>`, `"><img src=x onerror=alert(1)>`) — it must
    render as literal text everywhere it is later displayed.
35. **Numbers and dates.** Numbers: 0, negative, decimals beyond the allowed precision,
    locale separators (`1.234,5`), scientific notation (`1e3`), leading zeros, very large values.
    Dates: 29 February in leap and non-leap years, the 31st of a 30-day month, month and year end, a
    date across a DST change, a user time zone different from the server's, start after end.
    Uniqueness: re-create an existing name or email differing only in case or trailing space.
36. **Input methods.** Paste values (with surrounding whitespace, formatted numbers,
    card numbers with spaces), drag-and-drop where offered, and autofill: `autocomplete` attributes
    (`email`, `new-password`, `one-time-code`, address parts) are present and correct.
37. **Validation** appears on blur or submit — not on the first keystroke — next to the
    right field, says how to fix it, clears as soon as the value is valid, and the entered values
    survive a failed submit (a form that wipes itself on error is a Major defect). Then **bypass the
    client**: remove `required` / `maxlength` / `disabled`, or change a hidden or select value with
    `browser_evaluate`, and submit. The server must reject what the UI blocks.
    - **Why:** client-only validation is not validation — it is a UI hint.
38. **Submit.** Double-click and Enter-spam must not create duplicates; the button shows
    progress and is disabled while in flight; Enter submits single-purpose forms.
39. Selects, date pickers, file uploads (`browser_file_upload`: wrong type, oversized,
    empty, duplicate name), checkboxes and toggles whose visual state matches the saved state after
    reload.

**Navigation and session**
40. Back / Forward and reload at every step; deep link to an inner step or detail URL, logged in
    and logged out; a URL with a non-existent or foreign id (clean 404 / 403, not a crash or a
    leak). Back after a successful submit must not re-POST silently or show a stale "pending" state;
    a page restored from the back-forward cache must refresh data that changed since (cart,
    balance, auth state).
41. **Session end.** (a) Expiry mid-flow → login → returned to the same place with input
    kept where feasible. (b) After logout, press Back: pages with personal data are not shown
    (`Cache-Control: no-store`). (c) Save the session cookie before logout (rule 15), restore
    it after: the server must reject it. (d) Log out in one tab, act in another: login is required.
42. No dead links; external links open as intended; the active nav item is highlighted; page
    title and URL change per view.

**Resilience**
43. Offline and failing API on each submit (rule 16): clear message, retry possible,
    no half-applied state, no infinite spinner, recovery when the network returns.
44. Slow responses (rule 17): progress is shown, conflicting input is prevented
    meanwhile, nothing times out silently, and out-of-order responses (search-as-you-type) never
    display stale results.

**Layout, platform, and output**
45. **Breakpoints.** Minimum set for any change: 320 (WCAG reflow floor), 390×844, and
    1366×768. Full set for flows in scope of a release or Boy Scout mode: add 360×800, 844×390
    (phone landscape), 768×1024, 1024×768, 1920×1080, and 1 px either side of every breakpoint the
    app's CSS defines. At each size: no horizontal scroll (except data tables, maps, diagrams),
    nothing overlapped or clipped, menus reachable, fixed headers not covering content, modals not
    taller than the viewport, multi-column forms collapsed to one column on narrow screens — and the
    layout pass of §7.
    - **Why:** layout bugs cluster at the exact width where a media query flips, and 320 px is where
      WCAG 1.4.10 requires content to reflow.
46. Dark mode, reduced motion, forced colors, and 200 % zoom (§5) if the app supports them.
47. **Localisation and formatting.** Every supported locale switches fully (no untranslated
    strings, no raw keys like `checkout.title`), longer translations fit, dates / numbers /
    currencies use locale formats, time zones are shown or unambiguous, and sorting follows the
    locale's collation.
48. **Outputs.** Exported or downloaded files (CSV, PDF) open correctly, respect the current
    filters, and escape formula-leading cells (`=`, `+`, `-`, `@`). Print preview via
    `browser_emulate_media` with `media: "print"` shows content, not navigation chrome.

**Browser-visible security smells** — observe and report; defer deep analysis to the project's
security review.
49. Tokens or PII in URLs, in `localStorage`, or in responses the screen does not need;
    secrets or stack traces in error responses; an authorised-only page rendering briefly before
    redirecting; a foreign resource reachable by editing an id in the URL; a hidden, disabled, or
    status field whose tampered value the server accepts (rule 37). Probe only with the
    session's own test accounts, never against real users' data.

## 5. Accessibility

Target **WCAG 2.2 AA** unless the project states otherwise (EU consumer-facing products fall under
the European Accessibility Act → EN 301 549, which embeds WCAG 2.1 AA; 2.2 AA is a superset). The
snapshot is Playwright's computed ARIA tree — a close proxy for what assistive technology receives,
not a screen reader. Tag every finding with its success criterion and level; label `best-practice`
items as such, never as WCAG failures.

50. **Run axe-core on every screen and every state** (open modal, error state, empty state) via
    `browser_run_code_unsafe`; report `violations` and hand-check `incomplete`:

     ```js
     async page => {
       const src = await (await page.request.get('https://cdn.jsdelivr.net/npm/axe-core@4.13.0/axe.min.js')).text();
       await page.evaluate(src);
       return page.evaluate(async () => {
         const r = await axe.run(document, { runOnly: { type: 'tag', values:
           ['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa', 'wcag22aa', 'best-practice'] },
           resultTypes: ['violations', 'incomplete'] });
         const slim = a => a.map(v => ({ id: v.id, impact: v.impact, help: v.help, n: v.nodes.length,
           targets: v.nodes.slice(0, 5).map(x => x.target.join(' >>> ')) }));
         return { violations: slim(r.violations), incomplete: slim(r.incomplete) };
       });
     }
     ```
    - **Why:** automated rules catch only about half of real issues — a clean scan is the floor,
      not a pass. Evaluating the source string (the mechanism `@axe-core/playwright` uses) avoids
      the CSP block that a `<script>` tag hits; cross-origin iframes are not scanned.
51. **Names, roles, states** (4.1.2, 1.1.1, 2.5.3): every control has an accessible name, the
    correct role, and its current state (`expanded`, `checked`, `selected`, `disabled`); meaningful
    images have alt text, decorative ones `alt=""` and are absent from the tree; the accessible name
    contains the visible label text.
52. **Keyboard-only walk** of each critical flow: everything operable (2.1.1); no trap
    (2.1.2); focus order meaningful (2.4.3); focus visible (2.4.7) and never fully hidden under
    sticky headers, cookie bars or chat widgets (2.4.11); a skip link or landmarks bypass repeated
    blocks (2.4.1); modals take focus inside, keep Tab inside, close on Escape, and return focus to
    the trigger if it still exists; custom widgets follow their ARIA Authoring Practices keyboard
    pattern; hover / focus popups are dismissable with Escape, hoverable, and persistent (1.4.13).
53. **Structure:** correct `<html lang>` (3.1.1); a unique, descriptive `<title>` per view
    (2.4.2); visual headings are real headings that describe their section (1.3.1, 2.4.6);
    landmarks exist. One `h1` and no skipped heading levels are best practice — report as minor.
54. **Forms:** visible labels (3.3.2); required fields indicated by more than colour;
    errors in text next to the field (3.3.1) with a fix suggestion (3.3.3), linked via
    `aria-describedby`, field marked `aria-invalid`, and **announced** — via `role="alert"` / a live
    region, or by moving focus to the error or an error summary; personal-data fields carry the
    right `autocomplete` (1.3.5); data already given is not asked again in the same process
    (3.3.7); login allows paste and password managers and has no puzzle CAPTCHA without an
    alternative (3.3.8).
    - **Why:** `aria-describedby` associates a message, it does not announce it — an error that
      appears silently is invisible to a screen-reader user who just pressed Submit.
55. **Status messages** (toasts, "Saved", result counts, cart updates) live in a
    `role="status"` / `role="alert"` / `aria-live` region that exists in the DOM before its text
    changes (4.1.3).
56. **Contrast:** text 4.5:1; large text (≥ 24 px, or ≥ 18.66 px bold) 3:1 (1.4.3); UI
    component boundaries, focus indicators, icons and chart parts 3:1 (1.4.11). Never round — 4.49:1
    fails. Prefer axe `color-contrast`; measure by hand only its `incomplete` items (text over
    images, gradients, transparency). State is never conveyed by colour alone (1.4.1).
57. **Reflow, zoom, text spacing:** at 320 px width no two-direction scrolling except data
    tables, maps, and diagrams (1.4.10); at 200 % zoom (or 640 px width) nothing is clipped (1.4.4);
    with the text-spacing override applied, nothing clips or overlaps (1.4.12):

     ```js
     () => { document.head.insertAdjacentHTML('beforeend', '<style id="qa-ts">*{line-height:1.5!important;' +
       'letter-spacing:.12em!important;word-spacing:.16em!important}p{margin-bottom:2em!important}</style>');
       return 'applied'; }
     ```
    Remove it afterwards (`document.getElementById('qa-ts').remove()`).
58. **Motion and modes:** content that moves on its own for more than 5 s can be paused
    (2.2.2); with `reducedMotion: "reduce"` non-essential animation stops; with
    `forcedColors: "active"` focus rings, borders, and icons stay visible; nothing forces a single
    orientation (1.3.4); every drag action has a single-pointer alternative (2.5.7).
59. **Target size:** AA minimum 24×24 CSS px, or no other target within a 24 px circle
    centred on it, with inline and equivalent-control exceptions (2.5.8). 44×44 is the AAA / platform
    recommendation (2.5.5, Apple HIG; Material uses 48 dp) — report shortfalls against it as UX
    findings, not WCAG failures.

## 6. UX evaluation

Judge the experience of each flow, not only its correctness — against all ten Nielsen heuristics.
Each finding names the heuristic or rule violated, the screen and step, the evidence (snapshot
excerpt, screenshot, measured number, failed walkthrough question) and the user impact. Findings
are expert predictions, not observed user behaviour: say so, and mark low-confidence ones for user
research.
- **Why:** a single evaluator finds only part of the problems, and a heuristic breach is not
  automatically a problem in context. Preference dressed up as a heuristic produces false positives
  that erode trust in the whole report.

60. **Visibility of system status.** Visible response to input within 0.1 s (pressed
    state, optimistic update); a busy indicator past ~1 s; past ~10 s a percent-done indicator and
    the freedom to do something else. The user always knows where they are (step indicator, active
    nav, page title).
61. **Match with the real world.** The user's vocabulary, not internal jargon, enum values,
    or database names (`status: PENDING_REVIEW_2`); units and currencies explicit; information in a
    natural, logical order.
62. **User control and freedom.** Cancel and Back exist and work at every step; destructive
    actions are undoable or confirmed; closing a modal never discards work silently. Confirm only
    rare, irreversible, high-stakes actions: the prompt names the object and the consequence, the
    buttons carry the action ("Delete 3 invoices" / "Keep"), never Yes / No, and there is no default
    choice. Confirmation on routine actions is a finding (users learn to click through it).
63. **Consistency and standards.** The same thing has the same name, icon, colour and
    position across screens; primary / secondary / destructive buttons are styled consistently;
    formats do not vary between screens. Platform and web conventions hold (logo goes home,
    underlined text is a link, cart top-right); a deviation needs a reason.
64. **Error prevention.** Constraints shown before the error (password rules, max size,
    formats); impossible options disabled with a visible reason; smart defaults reduce input.
65. **Recognition over recall.** Nothing from a previous step must be remembered;
    summaries before commit; recently used values offered.
66. **Flexibility and efficiency.** Frequent tasks have accelerators that do not get in
    novices' way: keyboard submit, bulk actions, remembered filters and sort, saved state. Their
    absence on high-frequency screens (admin tables, back office) is a finding.
67. **Aesthetic and minimalist design, visual hierarchy.** One clear primary action per
    screen; the most important information is seen first; related controls are grouped; nothing
    irrelevant or rarely needed competes with the task.
68. **Help users recover from errors.** Messages say what happened, why, and how to fix it,
    in plain language, next to the problem; they keep the user's input, appear only once the user
    has finished with the field, avoid blame words ("invalid", "illegal"), and offer the fix where
    possible. "Something went wrong", raw codes, and stack traces are findings.
69. **Help and documentation.** Where a task is not self-explanatory (complex fields, domain
    terms, fees), help is contextual — inline hint, tooltip, link to the exact doc — and absent where
    not needed. A help link that 404s or lands on a generic FAQ is a finding.
70. **Friction count per flow:** steps, screens, clicks, and fields to reach the goal. Flag
    fields that could be inferred or deferred, confirmations that add no safety, forced account
    creation before value (a top cause of checkout abandonment), and dead ends — screens with no
    obvious next action.
71. **Microcopy.** Button labels are verbs naming the outcome ("Save changes", not "OK");
    empty states explain what belongs there and offer a direct action to add the first item; tone is
    consistent; no typos, `Lorem ipsum`, `TODO`, or developer copy.
72. **Touch and mobile.** Targets per rule 59; inputs use the right keyboard type
    (`inputmode`, `type="email"`); nothing depends on hover; content readable without zoom.
73. **Perceived performance, measured.** On a fresh load, before interacting, run this with
    `browser_evaluate`. Good: LCP ≤ 2.5 s, CLS ≤ 0.1; poor: LCP > 4 s, CLS > 0.25. Lab numbers are
    one sample on one machine — report them as indicative, with viewport and throttling, never as
    field Core Web Vitals.

     ```js
     () => new Promise(done => {
       const r = { lcpMs: null, cls: 0 }; let win = 0, first = 0, last = 0;
       new PerformanceObserver(l => { const e = l.getEntries().at(-1); if (e) r.lcpMs = Math.round(e.startTime); })
         .observe({ type: 'largest-contentful-paint', buffered: true });
       new PerformanceObserver(l => { for (const e of l.getEntries()) {
           if (e.hadRecentInput) continue;
           if (win && e.startTime - last < 1000 && e.startTime - first < 5000) win += e.value;
           else { win = e.value; first = e.startTime; }
           last = e.startTime; r.cls = Math.max(r.cls, win); } })
         .observe({ type: 'layout-shift', buffered: true });
       setTimeout(() => done({ ...r, cls: +r.cls.toFixed(3) }), 3000);
     })
     ```
    For one slow interaction: before acting, run
    `() => { window.__qaEvt = 0; new PerformanceObserver(l => l.getEntries().forEach(e => window.__qaEvt = Math.max(window.__qaEvt, e.duration))).observe({ type: 'event', durationThreshold: 16, buffered: true }); }`,
    act, then read `window.__qaEvt`: over 200 ms is a finding, over 500 ms is poor. Also: skeletons
    over blank screens, no flash of wrong content (a logged-out header before the session resolves).
    - **Why:** LCP stops reporting at the first click, scroll, or key press — measured after
      interacting, the number is wrong.
74. **Clarity at commitment points** (payment, delete, submit, publish): the user sees
    exactly what will happen, what it costs — no surprise fees at the last step — and whether it is
    reversible.
75. **Grade every UX finding on Nielsen's 0–4 scale:** 0 — not a usability problem;
    1 — cosmetic, fix only if extra time is available; 2 — minor, low priority; 3 — major, important
    to fix, high priority; 4 — usability catastrophe, imperative to fix before release. Weigh
    frequency, impact (how hard to overcome), persistence (once or every time), and market impact
    together — there is no formula. A 3 or 4 needs evidence: a failed walkthrough question, a blocked
    flow, a measured threshold breach, or a standards violation — never preference.

## 7. Visual and layout quality

Screenshots are good for the overall impression but unreliable for pixel-level judgment — a 4 px
misalignment or an off-scale gap is invisible in a downscaled image. **Measure layout with
`browser_evaluate` (or `browser_snapshot` with `boxes: true`); use the screenshot to confirm what
the numbers say.** Run this pass on every form, card, table, and grid a flow touches, at the
breakpoints of rule 45. Report layout findings with the measured numbers ("label left
edge 24 px, input left edge 32 px"), never with "looks slightly off".

76. **Measure, don't eyeball.** Collect the geometry of one layout unit (narrow the selector to
    the container under test):

     ```js
     () => [...document.querySelectorAll('form label, form input:not([type=hidden]), form select, form textarea, form button, form [role=combobox], form [role=textbox]')]
       .filter(e => e.checkVisibility ? e.checkVisibility() : e.offsetParent !== null)
       .map(e => { const r = e.getBoundingClientRect(), f = n => +n.toFixed(1);
         if (!r.width || !r.height) return null;
         const id = e.name || e.htmlFor || e.id || (e.textContent || '').trim().slice(0, 20);
         return { el: `${e.tagName.toLowerCase()}${e.type ? ':' + e.type : ''}[${id}]`,
                  left: f(r.left), right: f(r.right), top: f(r.top), bottom: f(r.bottom),
                  w: f(r.width), h: f(r.height) }; })
       .filter(Boolean)
     ```
77. **Alignment.** Elements in one column share a left edge and, when full-width, a right
    edge; compare unrounded values — ≤ 1 px is subpixel noise, ≥ 2 px is a finding. Labels are placed
    the same way across the whole form (top-aligned labels complete fastest and are the rule on
    narrow screens); inline label + control pairs share a baseline; buttons and inputs in one row
    have equal height; icons are vertically centred with their text; numeric table columns are
    right-aligned (ideally `font-variant-numeric: tabular-nums`), text columns start-aligned.
78. **Spacing rhythm.** Gaps, margins, and paddings come from the project's spacing scale
    (commonly 4 / 8 px steps: 8 for layout, 4 for type and small elements). Off-scale values, or the
    same relationship spaced differently in two places, are findings. **Proximity:** a label sits
    closer to its own field than to the previous field (label-to-field gap < field-to-next-label
    gap) — otherwise the eye pairs it with the wrong control.
79. **Form composition.**
    - Sequential forms are single-column; two columns only for short, tightly related fields (first /
      last name, city / postcode, expiry / CVC).
    - **Field width reflects expected input length**: postcode, CVC, quantity short; email and
      street address long. Every field stretched full-width, or widths that vary randomly, are
      findings.
    - Every field has a persistent visible label — a placeholder is never the label.
    - Mark both required (`*`) and optional ("optional") fields, consistently, next to the label.
    - **Balance:** fields in a row have equal heights and aligned tops; no orphan field alone in a
      row beside an empty column; groups separated by a heading or a larger gap than within a group.
    - Help text, markers, and error messages sit in the same position relative to their field
      throughout, and an appearing error does not overlap the next field.
    - The primary action is aligned with the input column and placed after the last field; secondary
      actions are visually subordinate; destructive actions are separated from the primary one.
80. **Typography.** Inventory the type styles in use and compare with the project's type
    scale:

     ```js
     () => { const styles = new Map(), sizes = new Set(), issues = [];
       for (const e of document.querySelectorAll('body *')) {
         if (/^(SCRIPT|STYLE|NOSCRIPT|TEMPLATE|SVG)$/i.test(e.tagName) || (e.checkVisibility && !e.checkVisibility())) continue;
         const text = [...e.childNodes].filter(n => n.nodeType === 3).map(n => n.textContent).join('').trim();
         if (!text) continue;
         const c = getComputedStyle(e), fs = parseFloat(c.fontSize);
         const lh = c.lineHeight === 'normal' ? 'normal' : +(parseFloat(c.lineHeight) / fs).toFixed(2);
         sizes.add(fs);
         const k = `${c.fontFamily.split(',')[0]} ${fs}px/${lh} w${c.fontWeight}`;
         styles.set(k, (styles.get(k) || 0) + 1);
         if (text.length > 200) {
           const lines = Math.max(1, Math.round(e.getBoundingClientRect().height / (lh === 'normal' ? fs * 1.2 : lh * fs)));
           const cpl = Math.round(text.length / lines);
           if (cpl > 90 || (lines > 1 && cpl < 45) || (lh !== 'normal' && lh < 1.3)) issues.push({ tag: e.tagName, cpl, lh, sample: text.slice(0, 40) });
         } }
       return { sizes: [...sizes].sort((a, b) => a - b), styles: [...styles].sort((a, b) => b[1] - a[1]), issues }; }
     ```
    Findings: near-duplicate sizes (15 px next to 16 px), one-off styles, sizes off the type scale.
    More than ~6 sizes on one screen is a smell, not a rule. Body text: line-height about 1.2–1.5
    (flag < 1.3), 45–90 characters per line (flag > 80 in long reading text), text never touching
    its container edge.
81. **Visual balance and detail consistency.** Whitespace is distributed — no crammed block
    next to an empty one; cards in a grid row have equal heights; corner radii, borders, shadows, and
    icon sizes and strokes are consistent across components of the same kind. On wide screens (1920
    px) text blocks and forms are width-capped, not stretched edge to edge. In RTL locales, layout,
    alignment, and direction-implying icons mirror.
82. **Overflow and truncation.** `scrollWidth > clientWidth` or `scrollHeight >
    clientHeight` on a text element means clipped content; ellipsis-truncated text needs a way to see
    the full value (tooltip, expand, accessible name). Re-check after switching locale.
83. **Layout shift.** Reload with a cold cache and watch for shifts from font swap, late
    banners or cookie bars, and a scrollbar appearing (`scrollbar-gutter`); CLS above 0.1 (rule
    73) is a finding. Interaction states — hover, focus, active, disabled, loading, error — change
    appearance, not geometry: a thicker focus border that shifts neighbours, or a button whose width
    jumps when its label becomes a spinner, is a finding (compare boxes before and after).
    - **Why:** users click the wrong target when content moves under the pointer.
84. **Layering and sticky elements.** Sticky headers and footers never cover focused fields,
    anchor targets, or the last list item; menus, tooltips, and date pickers are not clipped by an
    `overflow: hidden` ancestor; modals sit above toasts and headers consistently.
85. **Images.** Aspect ratio preserved (`naturalWidth / naturalHeight` vs the rendered box);
    blurry when `naturalWidth < renderedWidth × min(devicePixelRatio, 2)`; missing `width` /
    `height` or `aspect-ratio` causes layout shift on load.
86. **Design-system conformance.** If the project ships design tokens, a component library,
    or design files, compare computed colours, spacing, radii, and font sizes with them. Off-token
    values and hard-coded colours are findings; a hand-rolled control duplicating a library component
    is a finding.
87. **Grade layout findings on the scale of rule 75.** Pure cosmetics are usually
    1; misalignment or spacing that breaks grouping or reading order, a form layout that makes users
    skip or mis-pair fields, and overlap that hides content are 2–3. Text-spacing, reflow, and
    contrast failures are WCAG failures (§5), not cosmetics.

## 8. Exploration and product opportunities

88. **Explore throughout, not only at the end.** Pick tours by charter risk, each time-boxed
    (about 10–20 minutes of actions):
    - *money* — the features users pay for or are sold on;
    - *FedEx* — follow one piece of data from input through every screen, export, and email;
    - *back alley* — least-used settings and admin pages;
    - *saboteur* — interrupt, double-submit, conflicting edits, starve resources (rules 16,
      17);
    - *intellectual* — the hardest inputs and questions the feature claims to handle;
    - *garbage collector* — spot-check every item in a set: all menu entries, all error messages,
      all email templates;
    - *supermodel* — surface only: visuals, copy, layout (§7).

    Then a **product-elements sweep** for what the flows missed: data lifecycle, time (timeouts,
    time zones, date rollovers, concurrent sessions), platform (a second browser engine, blocked
    third-party scripts), and claims (help text, tooltips, and marketing copy tested as promises).
89. **When a defect is found, hunt its family:** the same component, validator, or API call
    elsewhere, *and* the same root-cause pattern (one unescaped field → every rendered user-supplied
    field; one missing loading state → every async view). With code access, find reuse sites by
    searching the code rather than guessing.
90. **Charter vs opportunity.** An important-looking off-charter problem may be pursued —
    log it as opportunity work. If most time goes to opportunity work, rewrite the charter to match
    what was actually done. If the charter cannot be fulfilled (environment down, missing access),
    stop and report the obstacle instead of drifting. Stop when the charter's risks are covered or
    new tests stop producing new information; otherwise state what remains when the budget runs out.

**Product opportunities** — features that are missing rather than broken. Good testers notice
what the product *should* let users do and does not; an AI tester also tends to generate endless
wish lists. These rules keep the first and suppress the second.

91. **Hunt for gaps on every screen a flow touches**, with these questions:
    - **Shown but not usable** — data that is displayed but cannot be searched, filtered, sorted,
      copied, or followed (a *Status* column with no status filter; an order id that is not a link
      to the order).
    - **Repetition without leverage** — an action done one item at a time that users plausibly
      need in bulk; a value re-typed that could be remembered or defaulted.
    - **No way back** — an operation that cannot be undone, reverted, or duplicated-and-edited.
    - **Forgotten context** — filters, sort, page size, or view mode lost on navigation or reload.
    - **Data locked in** — no export, print, or share where users need the data elsewhere.
    - **Dead-end navigation** — no path from an object to its related objects (customer → orders).
    - **Silent events** — something important happens (status change, failure, threshold) and the
      user must check manually instead of being told.
    - **Empty states and first use** without a next step or a sensible starting template.
92. **An opportunity needs evidence, like a bug needs an oracle.** Acceptable grounds:
    data present but not actionable (observed on screen); **inconsistency within the product** (the
    neighbouring table has the filter, this one does not); **observed friction** during the walk
    (a failed cognitive-walkthrough question, the same manual action repeated N times, a friction
    count, rule 70); **conventions of comparable products** that users will expect (search
    in a long list, saved filters, table export); **the product's own claims** (help text or copy
    promising more than exists). "It would be nice" without one of these is not reported.
    - **Why:** unbounded, ungrounded suggestions bury the real findings and make the whole report
      read as opinion.
93. **State the value, rank, and cap.** For each opportunity: who benefits (persona), how
    often the task occurs, what it saves or enables, the evidence, confidence (high / medium /
    low), and a rough effort guess clearly marked as a guess. Report **at most five** per session,
    ranked by value × confidence; list the rest as one-line mentions. Phrase each as an observation
    and a question for the product owner — the gap may be deliberate, on the roadmap, or against
    strategy, which the tester cannot know. Opportunities never change the quality verdict and are
    never implemented without a decision, even in Boy Scout mode (rule 95).

## 9. Boy Scout mode

Default mode is **scoped**: test the charter, and log — never silently drop — anything noticed
outside it. **Boy Scout mode** is enabled when the user asks for it ("boy scout", "polish
everything", "make it shine", "fix whatever you find"). Its goal is not "the ticket passes" but
"the product is left better than it was found": everything encountered is tested, reported, and —
when the codebase is available — fixed. Leave it better, not rewritten.

94. **Widen the net deliberately.** Beyond the charter flows, walk every flow reachable from
    them, every screen that shares their components, and each defect's family (rule 89).
    Nothing seen is dropped as "not in scope".
95. **Reproduce, then triage each finding before touching code:**
    - **Fix now** — a reproduced defect with a local, revertable fix that needs no product decision.
      Split these into **tidyings** (copy, typos, alignment, spacing, accessible names, token
      conformance — presentation and structure only) and **defect fixes** (console errors, missing
      states, duplicate requests, focus bugs — behaviour changes). Never mix the two kinds in one
      change.
    - **Propose** — anything that changes product behaviour or needs a decision: flow redesign, new
      steps or confirmations, pricing / money / auth / permission logic, data-model or API contract
      changes, feature removal, large rewrites — and every product opportunity (rule 93).
      Write it with evidence and a recommended option; do not implement without the user's go-ahead.
96. **Fix at the root, one change per defect.** Fix the component, validator, or endpoint
    responsible — not a CSS override or a per-page patch over a shared bug — following the project's
    conventions and existing components, no new libraries for a polish fix. Add or extend an
    automated test that fails before the fix and passes after it, at the lowest layer that can catch
    the bug. Never make a symptom disappear by swallowing the exception, silencing the console,
    disabling the validator, or deleting the test. Keep changes separable by concern so each can be
    reviewed and committed independently; git operations follow the normal rules — no commits
    unless the user asked.
    - **Why:** a fix without a failing-then-passing test comes back as a regression; a silenced
      symptom turns a visible bug into silent data loss.
97. **Confirmation, then regression.** Record the failure before fixing. After the fix,
    re-run the exact steps from the bug report (confirmation), then re-walk every flow the changed
    code touches and run the project's tests, lint, and type check (regression). A finding is closed
    only when both pass.
98. **Loop until clean, within the budget.** Stop when any one holds: (a) a full pass over the
    walked flows yields no new severity-3/4 or Critical / Major findings, no unexplained console
    errors, and no unexpected 4xx / 5xx (known third-party or pre-existing noise is listed in the
    report, not counted); (b) the charter budget is spent; (c) the work is blocked on a decision. In
    (b) and (c), list exactly what remains.
99. **Without code access**, Boy Scout mode still widens the net (rule 94) and delivers
    the full findings log as a prioritised backlog with fix suggestions.

## 10. Reporting and cleanup

100. **Bug report — one defect per report, written neutrally**, title describing the problem, not
    the fix:

     ```
     Title: <failure> in <where> when <condition>        (≤ 70 chars)
     Build/commit · Env · Browser · Viewport/device · Role
     Severity: Blocker | Critical | Major | Minor | Trivial   (impact — set by the tester)
     Priority (suggested): P1–P4                            (urgency — the owner decides)
     Reproducibility: always | N of M | once
     Oracle: <claim / product consistency / standard … that says this is wrong>
     Preconditions:
     Steps (minimal, numbered):
     Expected: / Actual:
     Scope: also occurs in … / does NOT occur when …
     Worst consequence: <data loss, money, lockout, …>
     Workaround: <none | …>
     Evidence: console excerpt · request + status + payload excerpt · screenshot
     Suspected cause: <file:line, if the code was inspected>
     ```
    Never mark a bug reproducible without having reproduced it. Minimise the steps (isolate), look for
    the worst consequence (maximise), and check where else it occurs and where it does not
    (generalise) before filing.
    - **Why:** unverified or unminimised reports get closed as "cannot reproduce"; a report without
      scope gets a fix for one instance only.
101. **UX and layout finding:** screen / flow step, heuristic or rule violated (§6, §7),
    evidence (measured numbers for layout), user impact, severity 0–4, concrete recommendation,
    confidence.
102. **Opportunity:** the gap in one line, where it was observed, evidence type and
    evidence (rule 92), who benefits and how often, expected value, confidence, rough
    effort (a guess), and the question for the product owner.
103. **Accessibility finding:** success criterion and level (or `best-practice`), element
    and state, evidence (axe rule id, snapshot excerpt, measurement), user impact, fix.
104. **Report skeleton — the three-part testing story:**

     ```
     1. Verdict — product status: 2–3 lines on whether the flows under test work, top risks
     2. Findings by severity, then priority: bugs · accessibility · UX · layout · issues (open questions)
     3. How it was tested: charter · env/build · coverage matrix (flow × variant × breakpoint →
        pass / fail / not tested) · oracles and tools used
     4. How good the testing was: not tested and why · obstacles and testability problems ·
        residual risks · confidence (low / medium / high, one line why)
     5. Boy Scout: fixes applied (finding → root cause → files → test added → confirmation and
        regression result) · proposals awaiting a decision
     6. Product opportunities: top ≤ 5 by value × confidence, then one-line mentions
     7. Outlook: recommended next charters, ranked by risk
     8. Cleanup: test data created / removed · processes stopped
     ```
105. **Explicitly list what was not tested** and why (no credentials for a role, payment
    sandbox unavailable, environment down). Never write "everything works" — write what was verified.
    A report that hides its gaps is worse than a short one.
106. **Clean up:** `browser_close` when the session ends; stop any dev server or background
    process started for the session; reset throttling and remove `page.route` interceptions and
    injected styles; remove or list the test data created (rule 5). Playwright MCP writes
    snapshots, console logs, and screenshots into `.playwright-mcp/` under the workspace root —
    delete it (or confirm it is git-ignored) so session artefacts never land in a commit.

## When applying these rules

- **Be opinionated about footguns:** rule 7 (no Playwright MCP, no session — ask for the
  restore, never substitute), rule 2 (no oracle, no bug), rule 4 (state-changing
  actions against live systems), rule 11 (console with `all: true` + network after every
  step — the single biggest source of missed defects), rule 12 (fills skip key handlers), rule
  15 (HttpOnly cookies), rule 16 (Service Workers bypass routes), rule 24 (verify
  the outcome, not the toast), rule 27 (lost updates), rule 37 (client-only
  validation), rule 50 (a clean scan is the floor), rule 54 (silent errors), rule
  73 (measure before interacting), rule 76 (measure layout, don't eyeball), rule 95
  (tidyings mixed with behaviour changes), rule 96 (masked symptoms, fixes without a failing
  test), rule 100 (unverified reproducibility), rule 92 (no evidence, no
  opportunity — wish lists bury real findings), rule 105 (report what was not tested).
- **Scale the session to the change and its risk:** a one-line UI change in scoped mode deserves
  the functional pass on the touched screen, its flow's happy and error variants, the minimum
  breakpoint set, an axe scan, and the console / network check — not every tour. Money, auth, and
  data-loss flows get the full treatment.
- **Read the project first:** a QA checklist, design system, type or spacing scale, supported
  browser and breakpoint list, or accessibility target in the repo overrides the defaults here.
  Reuse existing components when fixing.
- **Cross-skill awareness:** `manual-qa-toolkit` for story analysis, test-case design techniques,
  and risk assessment before the session; the project's security review for anything rule
  49 surfaces; `pytest-best-practices` for regression tests of backend defects;
  `git-workflow-best-practices` for splitting Boy Scout fixes into commits.
