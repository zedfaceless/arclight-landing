# Arclight — landing page prototype

A single-file marketing site for **Arclight**, a fictional revenue-intelligence
product for subscription businesses. Markup, styles and behaviour all live in
[`index.html`](./index.html): no build step, no dependencies, no network
requests except the two Google Fonts.

```
open index.html          # macOS
xdg-open index.html      # Linux
```

Each section is written as a self-contained block so it can be lifted into a
React/Next.js component with the same boundary, class names and hooks — see
[Porting to React](#4-porting-to-react).

---

## 1. Design system

Tokens are CSS custom properties on `:root`; change them there rather than in
individual rules.

| Token | Value | Used for |
| --- | --- | --- |
| `--bg` | `#06070A` | page background |
| `--surface` / `--surface-2` | `#0C0E14` / `#11141C` | cards, mock dashboard |
| `--line` / `--line-strong` | `rgba(255,255,255,.08)` / `.14` | hairlines, card borders |
| `--text` / `--muted` / `--muted-2` | `#F5F7FC` / `#9AA3B5` / `#6B7488` | type ramp |
| `--violet` `--blue` `--mint` | `#7C5CFF` `#4C8DFF` `#3FE0B0` | brand + accents |
| `--grad` | violet → blue → mint at 135° | primary buttons, gradient text |
| `--danger` | `#FF6B7A` | form errors, "High" chips |
| `--header-h` | `72px` (`64px` ≤720px) | sticky header offset |
| `--ease` | `cubic-bezier(.22,.8,.24,1)` | transitions |

Type: **Inter** for headings and UI, **JetBrains Mono** for eyebrows, numbers
and labels. Display headings use `clamp()` sizing with tight tracking
(`-.045em`); body copy sits at `--muted` on `--bg` for AA contrast.

Layout: `.wrap` is the only content container (`max-width: 1180px`, 20px
gutters, 16px below 520px). Sections use `.section` with fluid
`padding-block: clamp(...)`, so vertical rhythm scales without extra
breakpoints.

Motion: `.reveal` elements fade and rise once when they enter the viewport
(`.is-in` is added by an observer). Everything is disabled under
`prefers-reduced-motion: reduce`, including the marquee, counters, spotlight
and drawer transitions.

---

## 2. Section → component map

Sections appear in DOM order. `id` values are the in-page anchors and double
as component boundaries.

| # | Anchor / selector | Eyebrow | React component | Notes |
| --- | --- | --- | --- | --- |
| 0 | `header.site-header#header` | — | `<SiteHeader />` | Fixed; adds `.is-stuck` after 12px. `#progress` is the 2px read bar |
| 1 | `section.hero#top` | `New` badge | `<Hero />` | H1, sub, dual CTA, trust row; `.hero__spot#spot` pointer glow |
| 1a | `.dash-wrap` | — | `<DashboardPreview />` | Static product mock: KPI cards, SVG area chart, leak queue |
| 2 | `section.logos` | — | `<LogoWall />` | `.marquee__track` is duplicated for the seamless loop |
| 3 | `section.stats` | — | `<StatBand />` | `[data-count]` values animate on first view |
| 4 | `section#problem` | The silent leak | `<Problem />` | Copy block plus a leak "receipt" card |
| 5 | `section#how` | How it works | `<HowItWorks />` | Three numbered steps |
| 6 | `section#features` | The platform | `<FeatureBento />` | Asymmetric bento grid, six features |
| 7 | `section#calculator` | ROI calculator | `<RoiCalculator />` | Live inputs, results panel, no-JS fallback values |
| 8 | `section#integrations` | Built on your stack | `<Integrations />` | Connector grid and "request a connector" line |
| 9 | `section#customers` | Proof, not promises | `<Testimonials />` | Three quotes with an outcome stat on top |
| 10 | `section#pricing` | Pricing | `<Pricing />` | Monthly/annual toggle, three plans |
| 11 | `section#faq` | Questions | `<Faq />` | Accordion plus sticky "still deciding" aside |
| 12 | `section#cta` | Your first audit is free | `<FinalCta />` | Email capture form |
| 13 | `footer.site-footer` | — | `<SiteFooter />` | Brand, four link columns, legal row, socials |
| 14 | `div#toast` | — | `<Toast />` | Fixed live region, bottom center |

Shared primitives worth extracting first: `Button` (`.btn`, `.btn--primary`,
`.btn--ghost`, `.btn--sm`, `.btn--lg`), `SectionHead` (`.section-head` +
`.eyebrow` + `.display`), `Card`, `Pill` and `GradientText` (`.grad-text`).

---

## 3. JavaScript hooks

The script is one IIFE at the bottom of the file; every behaviour is
independent, so a block can be removed without breaking the rest.

| Behaviour | Trigger / selector | State written | Notes |
| --- | --- | --- | --- |
| Sticky header | `#header`, `#progress` | `.is-stuck` on header; `transform: scaleX()` on the bar | Passive scroll listener inside `requestAnimationFrame` |
| Mobile drawer | `#burger`, `#drawer` | `.is-open` on both, `aria-expanded`, `aria-hidden`, `body.is-locked` | Esc closes; focus returns to the burger; links close it |
| Hero spotlight | `#spot` inside `#top` | `--mx`, `--my`, `.is-live` | Hover-capable pointers only, off with reduced motion |
| Reveal on scroll | `.reveal` | `.is-in` | Observer with `rootMargin: 0px 0px -8% 0px`, unobserves after firing |
| Count-up | `[data-count]` | text content | `data-decimals="1"`, `data-format="compact"` |
| Pricing toggle | `#billingMonthly`, `#billingAnnual` | `.is-on`, `aria-pressed`, `.num[data-monthly|data-annual]`, `.plan__annual[data-monthly-note|data-annual-note]` | Annual shows −20% and the billed-annually note |
| ROI calculator | `#mrr`, `#accounts`, `.chip-btn[data-rate]` | `#mrrOut`, `#accountsOut`, `#leakBig`, `#recovered`, `#cost`, `#net`, `#roi` | Model below; also repaints the slider fills via `--pct` |
| FAQ accordion | `.faq__item` > `.faq__q` | `.is-open`, `aria-expanded`, `aria-controls` | One panel at a time, animated height |
| Email form | `#auditForm`, `#email`, `#formMsg` | `.is-ok` / `.is-err`, `aria-invalid`, button label | Validates shape only, fakes 700ms send, writes `localStorage` |
| Toast | `#toast` | `.is-visible` | Auto-dismiss after 4.2s, `aria-live="polite"` |

### ROI model

```
ARR        = mrr * 12
leak       = ARR * rate + accounts * 18     // $18 of silent churn per account
recovered  = leak * 0.68                    // share Arclight typically recovers
plan cost  = ARR < 1.5M ? 2,868             // annual, 239 * 12
           : ARR < 8M   ? 8,628             // annual, 719 * 12
           :              24,000
net gain   = recovered - plan cost
ROI        = recovered / plan cost
```

`rate` comes from the leak-profile chips: annual contracts `2.8%`, mixed
`4.2%`, card-first `5.8%`. Defaults are `$120k` MRR, `850` accounts, mixed —
which is what the static HTML shows before JS runs.

---

## 4. Porting to React

A suggested Next.js (App Router) shape — one file per component, styles carried
over as a CSS module per section:

```
app/page.tsx                       // composes the sections in order
components/layout/SiteHeader.tsx
components/layout/MobileDrawer.tsx
components/layout/SiteFooter.tsx
components/sections/Hero.tsx       // + DashboardPreview.tsx
components/sections/LogoWall.tsx
components/sections/StatBand.tsx
components/sections/Problem.tsx
components/sections/HowItWorks.tsx
components/sections/FeatureBento.tsx
components/sections/RoiCalculator.tsx
components/sections/Integrations.tsx
components/sections/Testimonials.tsx
components/sections/Pricing.tsx
components/sections/Faq.tsx
components/sections/FinalCta.tsx
components/ui/{Button,SectionHead,Card,Toast}.tsx
lib/roi.ts                         // the model above as a pure function
lib/pricing.ts                     // plan data, monthly/annual numbers
content/copy.ts                    // every string, editable without JSX
```

Notes:

- **State.** `useState` for `billing`, `mrr`, `accounts`, `rate`, `openFaq`,
  `drawerOpen` and `formState`. The calculator output is a pure function of
  those values, so the result panel needs no effects; keep the `<output>`
  elements controlled and keep the `aria-live="polite"` wrapper.
- **Reveal and counters.** Replace the observers with a small `useInView` hook
  (`IntersectionObserver`, `once: true`) so the CSS contract
  (`.reveal` → `.is-in`) stays identical and no markup changes are needed.
- **Scroll effects.** Header shadow and progress are better as one scroll hook
  that writes a single CSS variable, applied with `transform: scaleX()`; keep
  the listener `{ passive: true }`.
- **The form is the only real integration point.** It currently fakes a 700ms
  send, writes `arclight:audit-email` to `localStorage` and shows the toast.
  Look for the `POST /api/audit` comment in the submit handler — that is the
  single place to add a fetch call, a pending state and server error copy.
- **Anchor scrolling.** `scroll-margin-top: 86px` on section targets assumes a
  72px header; keep that rule in the layout stylesheet so hash links keep
  clearing the sticky bar.

---

## 5. Accessibility

- Skip link (`.skip-link`) to `#main`, visible on keyboard focus.
- Landmarks: `header` → `nav[aria-label="Primary"]`, `main#main`, one `h1`,
  one `h2` per section, `footer`.
- The drawer and FAQ manage `aria-expanded` / `aria-hidden`; the drawer closes
  on Esc and restores focus to the toggle.
- Decorative layers (`.ambient`, `.grain`, grid, spotlight, marquee) are
  `aria-hidden`; the mock dashboard is presentational but its text stays
  readable to screen readers as plain content.
- Form messages are announced through `aria-live="polite"`, and
  `aria-invalid` is toggled on the input.
- Interactive elements are real `button` / `a` elements with visible focus
  rings; the only hover-only flourish is the spotlight.
- `prefers-reduced-motion` removes transforms, the marquee and counter tweening.

---

## 6. QA status

Checked in headless Chrome against the local file.

Layout:

- No horizontal overflow at 1440, 1280, 1100, 1024, 981, 900, 768, 600, 430,
  390, 375, 360, 340 or 320px.
- The header fits at every width; the header CTA is hidden below 400px, where
  it would otherwise collide with the menu button.
- Nav links collapse into the burger at 980px; the drawer's nav rows are 62px
  tall and the burger is 42×42px.
- Footer stacks to two columns of vertical links on phones; links keep a
  30px+ tap height, and the leak list in the hero dashboard swaps its Owner
  column for Impact at the same breakpoint.

Behaviour:

- Drawer opens and closes from the burger, any nav link, and Esc;
  `body.is-locked` is applied and focus returns to the toggle.
- Pricing toggle switches `299 / 899 → 239 / 719` and rewrites the billing
  notes.
- Calculator at defaults (`$120k`, `850`, mixed): `$75,780` leak, `$51,530`
  recoverable, `$2,868` plan, `$48,662` net, `18.0×` ROI. At `$500k` MRR with
  card-first: `$363,300` leak, `$8,628` plan, `$238,416` net, `28.6×`.
- FAQ keeps one panel open; the form rejects a malformed address and confirms
  a valid one with inline copy plus toast.
- No console errors; the HTML parses with no unclosed or mismatched tags, no
  duplicate IDs, and the stylesheet braces balance.

Known limits, by design:

- Copy, logos, testimonials, plan numbers and connector names are invented
  placeholder content.
- The dashboard, chart and sparkline are inline SVG and divs, not a chart
  library.
- Nothing is persisted server-side; `localStorage` only remembers the last
  submitted email.
- Fonts load from Google Fonts at runtime; self-host them before shipping.




