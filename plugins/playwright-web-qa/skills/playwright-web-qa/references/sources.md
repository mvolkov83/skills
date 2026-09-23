# Canonical sources behind `playwright-web-qa`

Every section of the skill was validated against the primary sources below (September 2026).
Rules paraphrase and adapt them to a single AI agent driving one Playwright MCP browser; where a
number or level is quoted (WCAG success criteria, Web Vitals thresholds, Nielsen's scale), it is
taken from the source, not invented. Consult the source when a rule's scope is unclear.

## §1 Charter, oracles, and safety — §8 Exploration and product opportunities

- James Bach, *Session-Based Test Management* (STQE, 2000) — charters, session sheets, notes /
  bugs / issues, on-charter vs opportunity work, debrief (PROOF).
  https://www.ida.liu.se/~TDDD04/labs/2020/exploratory_testing/stqe-sbtm.pdf
- James Bach, *Heuristic Test Strategy Model* v6.0 — risk-based testing, product elements
  (SFDIPOT), claims testing. https://www.developsense.com/resource/htsm.pdf
- Michael Bolton, *FEW HICCUPPS* — consistency oracles.
  https://developsense.com/blog/2012/07/few-hiccupps
- Cem Kaner, definition of exploratory testing. https://kaner.com/?p=46
- James Bach & Michael Bolton, *Exploratory Testing 3.0*. https://www.satisfice.com/blog/archives/1509
- Elisabeth Hendrickson, *Explore It!* (Pragmatic Bookshelf, 2013), ch. 2 — charter template
  "Explore <target> with <resources> to discover <information>".
- James Whittaker, *Exploratory Software Testing* (2009) — the tourist metaphor (money, FedEx, back
  alley, saboteur, intellectual, garbage collector, supermodel).
  https://www.techtarget.com/searchsoftwarequality/tip/Six-tours-for-exploratory-testing-the-business-district-of-your-application
- NN/g, *Jakob's Law of Internet User Experience* — users expect a product to work like the others
  they already use; the basis for the comparable-products evidence type for opportunities.
  https://www.nngroup.com/videos/jakobs-law-internet-ux/
- ISTQB Certified Tester Foundation Level v4.0, §4.4 — experience-based techniques, error guessing,
  exploratory testing.

## §2 Playwright MCP mechanics

- microsoft/playwright-mcp README — tools, snapshot mode, `--isolated`, `--user-data-dir`,
  `--storage-state`, `--device`, capabilities. https://github.com/microsoft/playwright-mcp
- Playwright docs — BrowserContext API, network (route / unroute, Service Worker caveat), best
  practices. https://playwright.dev/docs/api/class-browsercontext ·
  https://playwright.dev/docs/network · https://playwright.dev/docs/best-practices
- Live tool schemas of the Playwright MCP server (`browser_type.slowly`,
  `browser_console_messages.all`, `browser_network_requests.static`, `browser_snapshot.boxes`,
  `browser_find`, `browser_emulate_media`).

## §3 User flows — §4 Functional heuristics

- Elisabeth Hendrickson, James Lyndsay, Dale Emery, *Test Heuristics Cheat Sheet* — data type
  attacks, CRUD, Zero / One / Many, Some / None / All, Beginning / Middle / End, Goldilocks,
  interruptions, starvation, follow the data.
  https://www.ministryoftesting.com/articles/test-heuristics-cheat-sheet
- Cem Kaner, *An Introduction to Scenario Testing* — scenarios from object life histories,
  disfavoured users, system events. https://kaner.com/pdfs/ScenarioIntroVer4.pdf
- ISTQB CTFL v4.0, §4.2 — equivalence partitioning, 2- and 3-value boundary value analysis,
  decision tables, state transition testing.
- OWASP Web Security Testing Guide — SESS-06 logout, SESS-11 concurrent sessions, ATHN-06 browser
  cache, BUSL-04/05/06 process timing, function-use limits, workflow circumvention.
  https://github.com/OWASP/wstg
- Michael Hunter, *You Are Not Done Yet* — checklist of overlooked areas.
- Mike Andrews & James Whittaker, *How to Break Web Software* (2006) — client-side validation
  bypass, state attacks.
- NN/g, *Task Analysis*. https://www.nngroup.com/articles/task-analysis/
- web.dev / MDN, back-forward cache. https://web.dev/articles/bfcache

## §5 Accessibility

- W3C, WCAG 2.2 and its Understanding documents (1.3.1, 1.4.3, 1.4.10, 1.4.11, 1.4.12, 2.4.11,
  2.5.8, 3.3.8, 4.1.3 and others). https://www.w3.org/WAI/WCAG22/Understanding/
- W3C WAI, *Easy Checks*. https://www.w3.org/WAI/test-evaluate/easy-checks/
- W3C WAI-ARIA Authoring Practices Guide — keyboard patterns, modal dialog.
  https://www.w3.org/WAI/ARIA/apg/
- Deque axe-core API and `@axe-core/playwright` injection mechanism.
  https://github.com/dequelabs/axe-core · https://github.com/dequelabs/axe-core-npm
- WebAIM Million (2026) — most common detected failures. https://webaim.org/projects/million/
- European Accessibility Act / EN 301 549 v3.2.1 — legal baseline for EU consumer products.

## §6 UX evaluation

- NN/g, *10 Usability Heuristics*. https://www.nngroup.com/articles/ten-usability-heuristics/
- NN/g, *Severity Ratings for Usability Problems*.
  https://www.nngroup.com/articles/how-to-rate-the-severity-of-usability-problems/
- NN/g, *How to Conduct a Heuristic Evaluation*.
  https://www.nngroup.com/articles/how-to-conduct-a-heuristic-evaluation/
- NN/g, *Response Times: The 3 Important Limits*.
  https://www.nngroup.com/articles/response-times-3-important-limits/
- NN/g, *Error-Message Guidelines*, *Confirmation Dialogs*, *Empty States*, *Cognitive
  Walkthroughs*. https://www.nngroup.com/articles/error-message-guidelines/ ·
  https://www.nngroup.com/articles/confirmation-dialog/ ·
  https://www.nngroup.com/articles/empty-state-interface-design/ ·
  https://www.nngroup.com/articles/cognitive-walkthroughs/
- Ben Shneiderman, *Eight Golden Rules of Interface Design*.
  https://www.cs.umd.edu/users/ben/goldenrules.html
- ISO 9241-110:2020 (interaction principles) and ISO 9241-11 (usability definition).
- web.dev, *Web Vitals*, *LCP*, *CLS*, *INP*. https://web.dev/articles/vitals
- Baymard Institute, checkout UX research. https://baymard.com/blog/checkout-flow-ux-optimization

## §7 Visual and layout quality

- Baymard Institute — multi-column forms, field width, required / optional marking, mobile label
  position, inline validation. https://baymard.com/blog/avoid-multi-column-forms ·
  https://baymard.com/blog/form-field-usability-matching-user-expectations ·
  https://baymard.com/blog/required-optional-form-fields ·
  https://baymard.com/blog/mobile-form-usability-label-position ·
  https://baymard.com/blog/inline-form-validation
- NN/g — web form design, placeholders in form fields, Gestalt proximity.
  https://www.nngroup.com/articles/web-form-design/ ·
  https://www.nngroup.com/articles/form-design-placeholders/ ·
  https://www.nngroup.com/articles/gestalt-proximity/
- Luke Wroblewski, *Web Form Design* — label alignment, primary and secondary actions.
  https://www.lukew.com/ff/entry.asp?504 · https://www.lukew.com/ff/entry.asp?571
- Matthew Butterick, *Practical Typography* — line length, line spacing.
  https://practicaltypography.com/line-length.html
- WCAG 2.2 Understanding 1.4.8, 1.4.10, 1.4.12.
- web.dev — optimise CLS, font `size-adjust`, responsive images.
  https://web.dev/articles/optimize-cls
- Android window size classes (Material 3 breakpoints); spec.fm 8-pt grid; W3C Design Tokens
  Community Group format (draft); MDN `scrollbar-gutter`.
- StatCounter screen-resolution statistics (August 2026) — breakpoint selection.
  https://gs.statcounter.com/screen-resolution-stats

## §9 Boy Scout mode — §10 Reporting

- Robert C. Martin, *The Boy Scout Rule* (97 Things Every Programmer Should Know).
  https://github.com/97-things/97-things-every-programmer-should-know
- Martin Fowler, *Opportunistic Refactoring*, *Preparatory Refactoring*.
  https://martinfowler.com/bliki/OpportunisticRefactoring.html
- Kent Beck, *Tidy First?* (2023) — separate tidyings from behaviour changes.
- Google Engineering Practices, *Small CLs*.
  https://google.github.io/eng-practices/review/developer/small-cls.html
- Cem Kaner, *Bug Advocacy*; BBST RIMGEA / RIMGEN.
  https://kaner.com/pdfs/BugAdvocacy.pdf · https://bbst.courses/rimgen/
- ISO/IEC/IEEE 29119-3 — incident report and test completion report contents.
- ISTQB glossary — confirmation vs regression testing, severity vs priority.
- Mozilla, *Bug writing guidelines*. https://bugzilla.mozilla.org/page.cgi?id=bug-writing.html
- Michael Bolton, *Braiding the Stories* — the three-part testing story.
  https://developsense.com/blog/2012/02/braiding-the-stories
