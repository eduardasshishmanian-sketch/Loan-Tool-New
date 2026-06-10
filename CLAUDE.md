# CLAUDE.md

Guidance for AI assistants (and humans) working in this repository.

## What this is

A **single-page, static web app**: an interactive private loan agreement tool.
It lets a prospective lender model a loan to Eduardas Shishmanian — choose an
amount, term, and interest-payout style — see the interest projected as an
animated chart, read the auto-populated loan agreement, submit their details,
and simulate a repayment ("withdrawal") request. Both the details submission
and the repayment request are emailed via EmailJS.

There is **no backend, no build step, no framework, and no dependencies to
install.** The entire application is one file.

## Repository layout

```
.
├── index.html   # The entire application: HTML + inline CSS + inline vanilla JS
├── CNAME        # Custom domain for GitHub Pages: edo-loan-tool.hydeparkcommerce.co.uk
├── README.md    # One-line title only
└── CLAUDE.md    # This file
```

Everything of substance lives in `index.html` (~713 lines):
- **Lines ~11–185** — inline `<style>`. CSS custom properties (the cream/forest/gold
  palette) are defined in `:root`. Includes a `@media print` block that hides the
  interactive chrome so "Save / Print as PDF" produces a clean agreement.
- **Lines ~187–337** — markup: hero, **setup card** (amount, date, term slider,
  interest-mode toggle), **interest visual card** (stat tiles + SVG chart),
  **document card** (the agreement clauses + lender details form), and the
  **exit/withdrawal simulator card**.
- **Lines ~339–711** — inline `<script>`, vanilla JS (no modules, no bundler).

## Hosting & deployment

- Deployed via **GitHub Pages** from the default branch, served at the custom
  domain in `CNAME` (`edo-loan-tool.hydeparkcommerce.co.uk`).
- "Deployment" = merge to the Pages branch; GitHub serves `index.html` directly.
  There is no CI/CD pipeline, no compilation, no asset pipeline.
- External resources are loaded from CDNs at runtime:
  - Google Fonts: Bricolage Grotesque, Fraunces, Hanken Grotesk.
  - **EmailJS** browser SDK (`@emailjs/browser@4`), lazy-loaded on first form
    submit via `loadEmailJS()`.

## How to run / test locally

There is no test suite and no dev server requirement. To preview:

```bash
# Just open the file
xdg-open index.html        # or: open index.html (macOS)

# Or serve it (needed only if you want a real origin, e.g. for some browser APIs)
python3 -m http.server 8000   # then visit http://localhost:8000
```

Verify changes **by eye in a browser** — there is nothing to lint or compile.
Check: the term slider + ticks, the amount slider/input sync, the animated SVG
chart, the monthly-vs-accruing toggle, the live-updating agreement text, the
print/PDF output, and both EmailJS flows (details submit, repayment request).

## Key domain logic & conventions

The loan model is encoded directly in the JS — keep these consistent with the
agreement clauses in the markup, since the two must always agree:

- **Fixed interest rate:** `RATE = 0.07` (7%/yr), `mRate = RATE/12`. Referenced
  both in the chart math and in clause text — change in both places.
- **Term:** slider `18–42` months, step `6` (ticks at 18/24/30/36/42).
  `MAXTERM = 42`, milestones `MILES = [18,24,30,36,42]`.
- **Interest start:** one month after the start date (`addMonths(s, 1)`).
- **Lock-in / notice rules:** 18-month minimum lock-in; 3 months' written notice;
  earliest interest-preserving notice is **15 months** in (`addMonths(s, 15)`),
  so repayment lands at the 18-month mark. Notice earlier than 15 months returns
  principal within 3 months but **forfeits interest**. This logic lives in the
  `#trigger` click handler and must match clauses 2–4 in the document card.
- **Monthly interest** = `P * mRate`; **accruing balance** = `P * (1+mRate)^term`
  (monthly compounding). The chart's gold line compounds; the green line is the
  paid-monthly (flat) interest path; grey is the principal.

### Code style in `index.html`

- Plain ES (no transpilation). Terse, single-letter helpers: `$ = id => getElementById`,
  `fmt`/`fmtShort` for dates, `money(n, dp)` for `£` formatting, `addMonths` for
  date math. Reuse these rather than re-implementing.
- DOM elements are cached once in the `els` object and an `$('id')` helper;
  follow that pattern when adding fields.
- `render()` is the central recompute function — it reads inputs and updates all
  derived text/chart. Most input listeners simply call `render()`. New inputs
  that affect output should funnel through it.
- The SVG chart is hand-built (`buildChartSkeleton`, `chartModel`, `paintModel`,
  `updateChart`) with `requestAnimationFrame`-based tweening (`lerpModel`).
- Currency/dates use UK conventions throughout (`en-GB`, `£`, day-month-year).

### EmailJS integration

- Public key, service id, and template id are **hardcoded** in the script:
  `EMAILJS_KEY`, service `service_hnl6ni7`, template `template_1yhpyff`. These are
  EmailJS public identifiers (safe for client-side use) — but treat them as
  config: if they change, update both the submit handler and the repayment handler.
- Both flows send the **same template** with a `params` object; the repayment
  flow overloads fields (e.g. `request_type`, `lender_address`) to signal a
  repayment request. Keep the param keys in sync with the EmailJS template.

## Working agreements for changes

- **One file.** Prefer editing `index.html` in place. Don't introduce a build
  system, framework, package manager, or split the file unless explicitly asked —
  the zero-tooling, GitHub-Pages-friendly setup is intentional.
- **Keep numbers and prose in lockstep.** Any change to rate, term bounds,
  lock-in, or notice rules must be reflected in *both* the JS and the agreement
  clause text (and the exit-simulator copy).
- **Don't commit secrets.** The EmailJS ids here are public by design; never add
  private keys/tokens to this client-side file.
- **Preserve the print stylesheet** when restructuring markup — the "Save / Print
  as PDF" feature depends on the `@media print` rules hiding non-document UI.

## Git workflow

- Develop on the designated feature branch; commit with clear messages and push
  with `git push -u origin <branch>`. Do **not** open a pull request unless
  explicitly asked.
- Because `index.html` is served straight from Pages, anything merged to the
  Pages branch goes live — review visual/behavioral changes before merging.
