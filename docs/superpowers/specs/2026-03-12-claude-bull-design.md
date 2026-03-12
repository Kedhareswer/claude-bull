# Claude Bull — Design Specification

**Date**: 2026-03-12
**Status**: Approved for implementation
**Author**: Design sprint (brainstorm session)

---

## 1. Problem Statement

**Who**: Individual investors, finance students, and retail traders who want institutional-quality equity research without paying $10K+/year for Bloomberg or FactSet.

**What**: They spend hours manually reading SEC filings (10-K, 10-Q, earnings transcripts) to understand a business — then have no structured way to turn those notes into a defensible investment thesis.

**Where**: The research happens asynchronously over hours or days, typically on weekends or evenings.

**Why it matters**: Bad investment decisions often come from incomplete research. The tools that enforce rigorous, citation-backed analysis cost thousands per year and require professional training.

**Current state**: Manual — copy-paste SEC text into Google Docs, no citation trail, no structured framework, no bear case discipline.

### Long-Term Goal
"In 6 months, any retail investor can produce a Goldman Sachs–quality research memo on any US public company in under 2 hours, with every AI claim traceable to an exact SEC filing."

### Sprint Question
"Can we build a wizard-guided research workflow where AI drafts each memo section while enforcing citation discipline, and where users trust the output because they can verify every claim?"

---

## 2. Winning Solution

**Linear Wizard + Live Split-View Memo** (Decision matrix score: 22/28)

A 9-step wizard that walks users through a structured research framework. Each step:
1. Fetches relevant SEC filings via EDGAR API
2. Sends filing text to Claude with a strict citation-enforcement prompt
3. Streams the AI response with inline citation badges
4. Writes the result directly into a live memo visible in a split pane

The final output is a professional investment memo with 100% citation coverage — every claim links back to a specific filing paragraph.

---

## 3. Architecture

### Application Type
**Web app** — Next.js 14 App Router hosted on Vercel free tier.

Rationale: shareable via URL (portfolio value), zero install, automatic updates, works on any device.

### BYOK Trust Model
- User provides their Anthropic API key once during onboarding
- Key stored **encrypted in localStorage** — never sent to or stored by the server
- Claude API calls go: `browser → /api/claude route handler → Anthropic API`
- The route handler acts as a thin CORS proxy only — it does not log, store, or inspect the key
- SEC EDGAR data fetching happens server-side (public API, no key needed)
- Trust strip on onboarding screen explicitly communicates this model

### Data Flow
```
User enters ticker → Next.js server fetches SEC EDGAR filings
→ Filing text sent to Claude via client-side call through /api/claude proxy
→ Claude streams response with citations
→ Citations validated (every claim must reference a filing paragraph)
→ Memo section written to localStorage/Zustand
→ Final memo exported as markdown or PDF
```

### Screens (3)
1. **Onboarding** — API key setup + ticker entry
2. **Research Wizard** — 9-step progress sidebar + streaming AI output + source panel
3. **Memo Output** — final formatted investment memo with citation sidebar

---

## 4. Tech Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Framework | Next.js 14 App Router | SSR + API routes in one project; Vercel native |
| Language | TypeScript | Type safety for financial data structures |
| Styling | Tailwind CSS + shadcn/ui | Rapid, consistent UI with accessible components |
| State | Zustand | Lightweight, no boilerplate; research state across steps |
| AI streaming | Vercel AI SDK (`useChat`) | Built-in streaming + citation parsing hooks |
| Storage | localStorage + IndexedDB | BYOK — no backend DB needed; researchs persist locally |
| Hosting | Vercel free tier | Zero config deployment, edge network |
| Data | SEC EDGAR API (free, public) | 10-K, 10-Q, 8-K, earnings transcripts |
| Fonts | Inter (UI) + JetBrains Mono (data) | Professional terminal aesthetic |

---

## 5. The 9-Step Research Framework

| Step | Name | Filing Source | AI Task |
|------|------|---------------|---------|
| 1 | Business Overview | 10-K Item 1 | Describe business model, revenue streams, customer segments |
| 2 | Industry Scan | 10-K Item 1A, 7 | Map competitive landscape, TAM, industry tailwinds/headwinds |
| 3 | Financial Deep Dive | 10-K Item 8, 10-Q | Analyze 3-year revenue trend, margins, FCF, debt ratios |
| 4 | Moat Analysis | 10-K Item 1, 7A | Identify and score competitive advantages (1-5 scale) |
| 5 | Management Review | Proxy, 8-K | Assess leadership quality, compensation alignment, track record |
| 6 | Valuation Check | 10-K, market data | P/E, P/FCF, EV/EBITDA vs peers and history |
| 7 | Bear Case | All filings | **Mandatory adversarial analysis** — what kills this thesis? |
| 8 | Earnings Update | Latest 10-Q, 8-K | Reconcile current quarter vs thesis assumptions |
| 9 | Decision Journal | Synthesized | Final verdict (BUY / WATCH / AVOID) with conviction level |

**Step 7 is gated**: the system will not allow navigation to step 8 without completing an adversarial analysis.

---

## 6. Citation Discipline System

Every AI response must follow this contract:

**Prompt enforcement rule**: "You MUST cite the exact filing source for every claim. Format: [FILING TYPE, Section X.X, paragraph Y]. If you cannot find a supporting citation in the provided text, do not make the claim."

**Citation badge format**: `10-K §1.2 ↗` (inline badge in memo prose)
- Green badge = verified (citation maps to provided filing text)
- Clicking badge expands to show the exact filing paragraph in the source panel

**No citation = no claim rule**: If Claude cannot find a supporting paragraph, it outputs `[CITATION NEEDED]` rather than fabricating a claim. These are highlighted in amber for human follow-up.

---

## 7. Design System

### Color Tokens
```css
--bg-base: #070711        /* Near-black background */
--bg-surface: #0d0d1f     /* Card/panel surfaces */
--bg-elevated: #111122    /* Elevated elements */
--border: #1a1a2e         /* Default borders */
--border-strong: #2a2a4a  /* Prominent borders */

--primary: #4fc3f7        /* Sky blue — primary CTA, active states */
--primary-dim: #1a3a4a    /* Primary background tints */

--text-primary: #e0e8ff   /* Main text — slightly blue-white */
--text-secondary: #7080a0 /* Secondary/muted text */
--text-dim: #3a4a6a       /* Placeholder, disabled */

--green: #4ade80          /* Positive values, citations, BUY verdict */
--red: #f87171            /* Negative values, warnings, AVOID verdict */
--amber: #fbbf24          /* Warnings, CITATION NEEDED */
--purple: #a78bfa         /* Moat badges, special annotations */
```

### Typography Scale
```css
--font-ui: 'Inter', system-ui, sans-serif
--font-data: 'JetBrains Mono', monospace

--text-xs: 11px / 1.4
--text-sm: 13px / 1.5
--text-base: 14px / 1.6
--text-lg: 16px / 1.5
--text-xl: 20px / 1.4
--text-2xl: 24px / 1.3
```

### Spacing Scale
```css
--space-1: 4px
--space-2: 8px
--space-3: 12px
--space-4: 16px
--space-5: 20px
--space-6: 24px
--space-8: 32px
--space-10: 40px
```

---

## 8. Component Inventory

### Global
- `NavBar` — logo, ticker chip, progress indicator, settings icon
- `TrustStrip` — "Keys never leave your browser | SEC filings only | ~$0.40/analysis"

### Onboarding Screen
- `SetupCard` — 2-step form: (1) API key input + link to Anthropic console, (2) ticker search with autocomplete
- `ApiKeyInput` — masked input with show/hide toggle, validation against Anthropic key format
- `TickerSearch` — debounced EDGAR company search with exchange badge (NASDAQ/NYSE)
- `StartButton` — primary CTA, disabled until both steps complete

### Research Wizard Screen
- `StepSidebar` — 9 steps with done/active/pending states, progress bar, ticker chip
- `StepHeader` — current step name, filing source badge, estimated cost
- `StreamingOutput` — AI response with inline `CitationBadge` components
- `CitationBadge` — `[10-K §X ↗]` badge; expandable to show source paragraph
- `MetricsGrid` — financial snapshot grid (revenue, margins, FCF, debt/equity)
- `BearCaseAlert` — red-bordered alert card for step 7, required completion gate
- `SourcePanel` — collapsible right panel showing raw filing excerpts mapped to citations
- `StepNavigation` — prev/next buttons + skip option (except step 7 gate)

### Memo Output Screen
- `MemoHeader` — company name, exchange, ticker, verdict badge, last updated date
- `MemoSection` — each of the 9 sections with heading, prose, inline citations
- `VerdictBadge` — BUY (green) / WATCH (amber) / AVOID (red) + conviction stars
- `MetricsSidebar` — key metrics, source count, filing breakdown, completion ring
- `ExportBar` — "Copy as Markdown" and "Export PDF" actions

---

## 9. Screen Layout Specifications

### Onboarding (Single column, centered)
- Max-width 480px card centered in viewport
- Brand header with logo + tagline
- 2-step card: numbered steps with connecting line
- Trust strip below card

### Research Wizard (3-panel)
- **Left sidebar**: 240px fixed, steps list + progress
- **Main area**: flex-grow, topbar + streaming content
- **Right source panel**: 280px, collapsible on mobile
- Breakpoint 768px: hide source panel (accessible via tab)
- Breakpoint 600px: hide left sidebar (accessible via hamburger)

### Memo Output (2-panel)
- **Main memo**: flex-grow, scrollable prose
- **Right sidebar**: 300px fixed, metrics + citation index
- Breakpoint 768px: sidebar collapses to top summary bar

---

## 10. Key UX Decisions

1. **No login required for MVP** — research stored in localStorage keyed by ticker. Login is a v2 feature.
2. **Streaming output** — research generates in real-time so users see progress. No "loading" black box.
3. **Step gate on bear case** — users cannot skip adversarial analysis. This is a product philosophy decision, not a technical constraint.
4. **Cost transparency** — estimated token cost shown per step and in total (~$0.40 for full analysis).
5. **Citation-first design** — the source panel is always accessible, not hidden. Every citation badge is interactive.
6. **Mono font for numbers** — all financial figures use JetBrains Mono for visual distinction and readability.

---

## 11. Validated Prototype

A functional 3-screen HTML prototype was built and verified at `prototype/index.html`:
- All 3 screens render correctly at desktop widths
- Navigation between screens works via bottom tab bar
- Design tokens implemented as CSS custom properties
- Responsive breakpoints at 768px and 600px

Prototype confirms the design direction is buildable and visually coherent.

---

## 12. Out of Scope (v1)

- User authentication / multi-device sync
- Team collaboration / shared research
- Portfolio tracking / position management
- International filings (non-US companies)
- Historical research comparison
- Email/PDF export with professional formatting
- Price alerts or watchlist notifications

---

## 13. Success Criteria

| Metric | Target |
|--------|--------|
| Time to first memo | < 20 minutes for a ticker the user knows |
| Citation coverage | 100% of AI claims have a filing citation |
| Bear case completion rate | 100% (gated by system) |
| Task completion (usability test) | ≥ 4/5 users complete full 9-step flow |
| SUS score | ≥ 68 (above average) |
| "Would you use this?" | ≥ 4/5 users say yes |

---

## 14. Implementation Phases

**Phase 1 — Foundation**
- Next.js 14 project scaffold with TypeScript + Tailwind + shadcn/ui
- Design token system (CSS custom properties matching prototype)
- Zustand store for research state
- SEC EDGAR API integration (ticker → CIK → filings)

**Phase 2 — Core AI Loop**
- Claude API proxy route handler (`/api/claude`)
- API key encryption/storage in localStorage
- Streaming AI response with Vercel AI SDK
- Citation parser (extract `[FILING, §X.X, para Y]` from response)
- Citation badge components + source panel linkage

**Phase 3 — Research Wizard**
- 9-step wizard with sidebar progress tracker
- Step-specific prompts for each research section
- Bear case gate logic (step 7 completion required)
- Metrics grid with financial data extraction
- Per-step cost estimation

**Phase 4 — Memo Output**
- Final memo assembly from 9 step outputs
- Verdict badge + conviction rating UI
- Citation index sidebar
- "Copy as Markdown" export
- Source breakdown statistics

**Phase 5 — Polish & Deploy**
- Mobile responsive fixes
- Loading states and error boundaries
- Onboarding flow with validation
- Vercel deployment + custom domain (optional)
- README with BYOK setup instructions
