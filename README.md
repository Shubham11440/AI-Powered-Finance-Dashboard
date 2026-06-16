# Proton Finance — Wealth Curator Dashboard

> A premium AI-powered personal finance dashboard built with **Vite + React 19 + TypeScript 6**. Features real-time budget analytics, dynamic rule-based AI insights, animated spending composition, interactive transaction ledger with CSV export, and full light/dark theme switching with smooth CSS transitions.

---

## Table of Contents

- [Getting Started](#getting-started)
- [Architecture](#architecture)
- [Component Map](#component-map)
- [Design System](#design-system)
- [Custom Hooks](#custom-hooks)
- [Insights Engine](#insights-engine)
- [Analytics (GA4)](#analytics-ga4)
- [Performance](#performance)
- [SEO & Accessibility](#seo--accessibility)
- [Trade-offs & Decisions](#trade-offs--decisions)

---



## Getting Started

```bash
# 1. Install dependencies
npm install

# 2. Configure environment (optional — GA4 only)
cp .env.example .env.local
# Edit .env.local and set VITE_GA_MEASUREMENT_ID=G-XXXX

# 3. Start dev server (HMR, ~50ms cold start)
npm run dev
# → http://localhost:5173

# 4. Type-check (no emit)
npx tsc --noEmit

# 5. Production build
npm run build

# 6. Preview production bundle
npm run preview
```

### Environment Variables

| Variable | Required | Description |
|---|---|---|
| `VITE_GA_MEASUREMENT_ID` | No | GA4 Measurement ID (e.g. `G-XXXXXXXXXX`). If absent, all analytics calls silently no-op. |

---

## Architecture

### Stack

| Layer | Technology | Version |
|---|---|---|
| Build tool | **Vite** | ^8 |
| UI framework | **React** + TypeScript | ^19 / ~6 |
| Styling | **CSS Custom Properties** (design tokens) + inline `CSSProperties` | — |
| Icons | **lucide-react** | ^1.17 |
| Charts | **Recharts** (lazy-loaded behind `React.lazy`) | ^3 |
| Analytics | **Google Analytics 4** via `gtag.js` | — |
| Theme state | **React Context** + `useLocalStorage` | — |
| Linting | ESLint + typescript-eslint | ^10 / ^8 |

> **Note on Tailwind**: `tailwindcss` and `@tailwindcss/vite` are listed as dependencies but the dashboard is styled exclusively with CSS Custom Properties and inline styles. Tailwind is unused in component files — it can be safely removed in a future cleanup.

---

## Component Map

### Full Component Tree

```
src/main.tsx
└── <React.StrictMode>
    └── App
        └── ThemeProvider              ← CSS token cascade + meta theme-color
            └── ErrorBoundary          ← Class-based catch-all fallback
                └── AppShell
                    ├── DashboardLayout
                    │   ├── Sidebar                    ← Fixed, always-dark nav
                    │   │   └── NavItemButton ×5 (memo)
                    │   ├── <header> → Header
                    │   │   ├── GlobalSearch           ← useDebounce(300ms)
                    │   │   ├── ThemeToggle            ← Sun / Moon (lucide)
                    │   │   ├── NotificationBell
                    │   │   └── UserAvatar
                    │   └── <main id="main-content">
                    │       └── <Suspense fallback=ChartSkeleton>
                    │           └── <ActivePage>       ← switched by activePage state
                    │               ├── Dashboard (default)
                    │               ├── BudgetPage  (lazy)
                    │               └── InsightsPage (lazy)
                    └── MobileBottomNav
```

### Per-Page Breakdown

#### `Dashboard` (default page)

```
Dashboard
├── PageHeader (title + action buttons)
├── Row 1: MetricRow + ActiveAlertsPreview
│   ├── MetricRow → MetricCard ×4 (memo) [useFetch]
│   └── ActiveAlertsPreview → AlertItem ×N (memo)
├── Row 2: ProStrategyCard (blue gradient hero)
├── Row 3: PortfolioDonut | AIInsights | BudgetTracker
│   ├── PortfolioDonut (conic-gradient, no lib)
│   ├── AI Insights → InsightCard ×3 (memo) [generateInsights]
│   └── BudgetTracker → category progress bars (memo)
└── Row 4: SpendingComposition | RecentActivity
    ├── SpendingComposition (animated %-width bars)
    └── RecentActivity [useFetch + useLocalStorage]
        ├── FilterPillButton ×6 (memo)
        └── TransactionRow ×N (memo)
```

#### `BudgetPage` (lazy-loaded)

```
BudgetPage [useFetch]
├── Page Header + "Adjust Limits" button
├── Left column (65%)
│   ├── Budget Velocity (animated progress bar)
│   ├── Stats panel (Projected Surplus / Savings Efficiency)
│   └── Category Grid → CategoryBudgetCard ×N
└── Right column (35%)
    ├── BudgetStrategyCard (compact blue gradient)
    └── AlertsPanel → ActiveAlertsPreview
```

#### `InsightsPage` (lazy-loaded)

```
InsightsPage [useMemo → generateInsights]
├── PageHeader
├── ProStrategyCard (full-width hero)
└── Insight grid → InsightCard ×(0–3) [dynamic, aria-live]
```

### All Source Files

```
src/
├── main.tsx                              ← ReactDOM.createRoot entry
├── App.tsx                               ← ThemeProvider + ErrorBoundary + lazy routing
├── App.css                               ← (minimal, mostly superseded by globals.css)
├── index.css                             ← (minimal reset shim)
│
├── components/
│   ├── AIInsights/
│   │   ├── InsightCard.tsx               ← AI insight chip; memo
│   │   ├── ProStrategyCard.tsx           ← Full-width blue gradient; memo + analytics
│   │   └── BudgetStrategyCard.tsx        ← Compact blue gradient; memo + analytics
│   ├── Alerts/
│   │   └── AlertsPanel.tsx               ← Thin wrapper re-exporting ActiveAlertsPreview
│   ├── Budget/
│   │   ├── BudgetTracker.tsx             ← Category progress bars; memo
│   │   └── CategoryBudgetCard.tsx        ← Individual budget card; memo
│   ├── Cards/
│   │   ├── MetricCard.tsx                ← KPI card with shimmer skeleton; memo
│   │   └── MetricRow.tsx                 ← Fetches + renders 4 MetricCards
│   │   └── ActiveAlertsPreview.tsx       ← Scrollable alert list; AlertItem memo
│   ├── Charts/
│   │   ├── ChartSkeleton.tsx             ← Shimmer placeholder for Suspense fallback
│   │   ├── PortfolioSparkline.tsx        ← Recharts LineChart; lazy
│   │   ├── SectorAllocationChart.tsx     ← Recharts RadialBarChart; lazy
│   │   ├── SpendingBarChart.tsx          ← Recharts BarChart; lazy
│   │   └── index.tsx                     ← Lazy wrappers: LazySpendingBarChart etc.
│   ├── Header/
│   │   ├── Header.tsx                    ← Sticky top bar; theme toggle, search, alerts
│   │   └── PageHeader.tsx                ← Per-page title + optional actions slot
│   ├── Layout/
│   │   ├── DashboardLayout.tsx           ← Sidebar + main shell + skip-nav link
│   │   ├── DashboardLayout.css
│   │   ├── Sidebar.tsx                   ← Fixed 240px dark nav; NavItemButton ×5
│   │   ├── MobileBottomNav.tsx           ← Bottom tab bar (mobile)
│   │   └── MobileBottomNav.css
│   ├── Spending/
│   │   └── SpendingComposition.tsx       ← Animated category progress bars; memo
│   ├── Transactions/
│   │   ├── RecentActivity.tsx            ← Filter pills + CSV export table; useFetch
│   │   └── TransactionList.tsx           ← Transaction type definitions + row list
│   └── common/
│       └── ErrorBoundary.tsx             ← Class component; catches render errors
│
├── context/
│   ├── theme.ts                          ← ThemeContext def + useTheme() export
│   └── ThemeContext.tsx                  ← ThemeProvider (useLocalStorage-backed)
│
├── data/
│   └── mockData.ts                       ← All typed mock datasets + fetchBudget()
│
├── hooks/
│   ├── index.ts                          ← Barrel re-export
│   ├── useAnalytics.ts                   ← GA4 wrapper; 10 typed events + helpers
│   ├── useDebounce.ts                    ← useDebounce<T> + useDebouncedCallback<T>
│   ├── useFetch.ts                       ← useReducer async fetcher; 3-attempt retry
│   └── useLocalStorage.ts                ← pf-namespaced storage; cross-tab sync
│
├── pages/
│   ├── Dashboard.tsx                     ← Main dashboard; generateInsights wired
│   ├── Dashboard.css                     ← Dashboard-specific layout overrides
│   ├── BudgetPage.tsx                    ← Budget management; useFetch(fetchBudget)
│   └── InsightsPage.tsx                  ← Dynamic AI insights; useMemo + analytics
│
├── styles/
│   ├── tokens.css                        ← Dark + light CSS custom properties
│   ├── globals.css                       ← Reset, utilities, universal transitions
│   └── tokens.ts                         ← TypeScript mirror of design tokens
│
└── utils/
    ├── analytics.ts                      ← getGtag() guard + gtagEvent()
    ├── exportCSV.ts                      ← RFC-4180 CSV builder + download trigger
    ├── insightsEngine.ts                 ← Rule-based AI insights generator
    └── index.ts                          ← Barrel exports
```

---

## Design System

### Proton UI Kit · Atmospheric Palette

Two complete theme sets — toggled via `data-theme` attribute on `<html>`:

| Token | Dark (`#0D1117` base) | Light (`#F4F6FA` base) |
|---|---|---|
| `--color-bg-base` | `#0D1117` | `#F4F6FA` |
| `--color-bg-surface` | `#161B27` | `#FFFFFF` |
| `--color-bg-sidebar` | `#0B0F1A` (**always dark**) | `#1A1D2E` (**always dark**) |
| `--color-bg-elevated` | `#1E2435` | `#EEF1F8` |
| `--color-bg-input` | `#1A2030` | `#F0F2F7` |
| `--color-primary` | `#0058BE` | `#0058BE` |
| `--color-success` | `#00A86B` | `#00875A` |
| `--color-warning-light` | `#F5A623` | `#924700` |
| `--color-error` | `#D93025` | `#C5281C` |
| `--color-text-primary` | `#FFFFFF` | `#0D1117` |
| `--color-text-secondary` | `#8B92A5` | `#5A6275` |
| `--color-border` | `#1E2A3A` | `#E2E6EF` |

> **Sidebar design**: The sidebar uses hardcoded dark colours (`#0B0F1A` / `#0B0F1A`) in its `CSSProperties` object — it does **not** inherit theme tokens. This is intentional: a permanently-dark sidebar creates strong visual hierarchy against the theme-reactive content area, mirroring Bloomberg Terminal and Robinhood Pro conventions.

### Typography — Editorial Hierarchy (9-step scale)

| Token | Size | Weight | Usage |
|---|---|---|---|
| `--font-display-lg` | 2.75rem | 700 | Net worth, large financial numbers |
| `--font-display-md` | 2rem | 700 | KPI card values |
| `--font-display-sm` | 1.5rem | 600 | Budget velocity amount |
| `--font-heading-lg` | 1.25rem | 600 | Section titles, card headers |
| `--font-heading-md` | 1rem | 600 | Sub-headers |
| `--font-heading-sm` | 0.875rem | 600 | Compact headers |
| `--font-body-md` | 0.875rem | 400 | Body text (14px) |
| `--font-label-md` | 0.75rem | 500 | Labels, badges (12px) |
| `--font-label-sm` | 0.6875rem | 500 | Status chips (11px) |

Typeface: **Inter** (Google Fonts, weights 400/500/600/700).

### Bento Surface Logic

```
Page: --color-bg-base
  └── Card: --color-bg-surface
        └── Hover / elevated state: --color-bg-elevated
              └── Input fields: --color-bg-input
```

### Theme Transition

`globals.css` applies a universal rule so every element transitions smoothly when `data-theme` flips:

```css
*, *::before, *::after {
  transition:
    background-color 0.2s ease,
    border-color     0.2s ease,
    color            0.1s ease;
}
```

`transform` and `opacity` are intentionally excluded — buttons, skeletons, and chart animations remain snappy.

---

## Custom Hooks

All hooks live in `src/hooks/` and are barrel-exported from `src/hooks/index.ts`.

### `useLocalStorage<T>(key, initialValue)`

**File**: `hooks/useLocalStorage.ts`

Type-safe localStorage with:
- Automatic `pf-` namespace prefix (all keys stored as `pf-{key}`)
- Cross-tab synchronisation via the `storage` event
- Updater-function support (`setValue(prev => ...)`)
- `reset()` to remove the key and restore the initial value

```ts
const [theme, setTheme, resetTheme] = useLocalStorage<Theme>('theme', 'dark')
// stored under "pf-theme"
```

**Used in**: `ThemeContext` (theme persistence), `RecentActivity` (filter state `pf-tx-filter`).

---

### `useAnalytics()` + `ANALYTICS_EVENTS`

**File**: `hooks/useAnalytics.ts`

GA4 event wrapper. All component analytics calls go through typed helpers — never raw `gtag()`. Dispatch is guarded by `getGtag()` in `utils/analytics.ts`:  if `window.gtag` is absent (ad-blocker, missing env var, SSR), calls silently no-op.

```ts
const { trackEvent, trackPageView, trackSearch, trackCTAClick,
        trackFilterClick, trackAlertDismiss, trackCSVExport } = useAnalytics()
```

| Helper | GA4 Event | Parameters |
|---|---|---|
| `trackPageView(path, title?)` | `page_view` | `{ page_path, page_title }` |
| `trackSearch(query)` | `search_used` | `{ query }` |
| `trackCTAClick(label, id, type?)` | `strategy_executed` | `{ cta_label, insight_id, insight_type }` |
| `trackFilterClick(cat, page)` | `filter_clicked` | `{ category, page }` |
| `trackAlertDismiss(id, sev)` | `alert_dismissed` | `{ alert_id, severity }` |
| `trackCSVExport(rowCount)` | `export_csv_clicked` | `{ row_count }` |
| `trackEvent(name, params?)` | _(any)_ | _(escape hatch)_ |

**Call sites** (all 10 `ANALYTICS_EVENTS` covered):

| Event constant | Fired in |
|---|---|
| `PAGE_VIEW` | `App.tsx` (initial load), `InsightsPage` |
| `SEARCH_USED` | `Header` (debounced 300ms) |
| `FILTER_CLICKED` | `RecentActivity` (category pill) |
| `STRATEGY_EXECUTED` | `ProStrategyCard`, `BudgetStrategyCard` |
| `REVIEW_AUDIT_CLICKED` | `ProStrategyCard` ("Review Audit") |
| `ALERT_DISMISSED` | `ActiveAlertsPreview` (X button) |
| `TRANSACTION_FILTERED` | `RecentActivity` (filter panel) |
| `THEME_TOGGLED` | `Header`, `Sidebar` (both have toggle) |
| `BUDGET_LIMIT_ADJUSTED` | `BudgetPage` ("Adjust Limits") |
| `EXPORT_CSV_CLICKED` | `RecentActivity` ("Export CSV") |

---

### `useDebounce<T>(value, delay?)` / `useDebouncedCallback<T>(fn, delay?)`

**File**: `hooks/useDebounce.ts`

Two exports:
- `useDebounce` — returns a debounced copy of a reactive value (value-based)
- `useDebouncedCallback` — returns a stable debounced function (keeps fn in a ref, safe as dependency)

Both clean up pending timeouts on unmount.

```ts
// Value debounce (used in Header search)
const debouncedQuery = useDebounce(rawQuery, 300)

// Callback debounce
const handleResize = useDebouncedCallback((w: number) => setWidth(w), 150)
```

**Used in**: `Header` (search analytics debouncing).

---

### `useFetch<T>(fetcher, deps?)`

**File**: `hooks/useFetch.ts`

`useReducer`-based async data fetcher with:
- **4-action state machine**: `FETCH_START → FETCH_SUCCESS | FETCH_ERROR | RESET`
- **3-attempt exponential backoff**: delays of 500ms → 1000ms → 2000ms
- **Stale-request protection**: request ID ref prevents out-of-order updates
- **Cleanup on unmount**: `isMountedRef` prevents state updates after unmount
- **`refetch()`** for manual re-trigger

```ts
const { data, loading, error, refetch } = useFetch<BudgetPageData>(fetchBudget)
```

**Used in**: `MetricRow` (dashboard KPI cards), `RecentActivity` (transaction list), `BudgetPage` (budget data).

---

## Insights Engine

**File**: `src/utils/insightsEngine.ts`

`generateInsights(transactions, budget)` — pure function, no side effects.

### Detection Rules

| Priority | Severity | Rule | Output example |
|---|---|---|---|
| 1 | `critical` | Any category `spent > limit` | "Entertainment budget exceeded" |
| 2 | `warning` | Entertainment `spent/limit > 85%` (not yet exceeded) | "Entertainment threshold approaching" |
| 3 | `warning` | `> 2` subscription transactions detected | "Cancel 3 inactive services, save $103/mo" |
| 4 | `info` | Dining transactions `> 5/week` average | "Dining spending 40% higher than average" |
| 5 | `info` | `savingsEfficiency > 90` | "Savings on track: 94.2% efficiency" |

Returns **max 3** insights sorted by severity (`critical → warning → info`). Subscription detection uses a keyword list of 17 known services. Dining detection matches `category === 'food'` or merchant name keywords.

**Used in**: `Dashboard` (AI Insights panel, `useMemo`), `InsightsPage` (insight grid, `useMemo`).

---

## Analytics (GA4)

### Setup

`gtag.js` is injected in `index.html` with `VITE_GA_MEASUREMENT_ID` substituted at build time. The inline init script guards `gtag('config', ...)` behind `typeof window.gtag === 'function'` to handle ad-blocker scenarios gracefully.

**`src/utils/analytics.ts`** — `getGtag()` returns `null` if:
- `window` is undefined (SSR)
- `window.gtag` is not a function (script blocked / env var missing)

No polyfill fallback — events simply don't queue, preventing phantom events.

### Event Table

| User Action | Event | Parameters |
|---|---|---|
| App loads | `page_view` | `{ page_path, page_title }` |
| Types in search (settled) | `search_used` | `{ query }` |
| Clicks category filter | `filter_clicked` | `{ category, page }` |
| "Execute Strategy" | `strategy_executed` | `{ cta_label, insight_id, insight_type }` |
| "Review Audit" | `review_audit_clicked` | `{ insight_id }` |
| Dismisses alert | `alert_dismissed` | `{ alert_id, severity }` |
| Filters transactions | `transaction_filtered` | `{ page }` |
| Toggles theme | `theme_toggled` | `{ newTheme }` |
| "Adjust Limits" | `budget_limit_adjusted` | `{}` |
| "Export CSV" | `export_csv_clicked` | `{ row_count }` |

---

## Performance

### Code Splitting

| Split point | Strategy | Fallback |
|---|---|---|
| `BudgetPage` | `React.lazy` | `ChartSkeleton` via `Suspense` |
| `InsightsPage` | `React.lazy` | `ChartSkeleton` via `Suspense` |
| `SpendingBarChart` | `React.lazy` in `Charts/index.tsx` | `ChartSkeleton(200)` |
| `PortfolioSparkline` | `React.lazy` in `Charts/index.tsx` | `ChartSkeleton(60)` |
| `SectorAllocationChart` | `React.lazy` in `Charts/index.tsx` | `ChartSkeleton(200)` |

All 5 lazy chunks confirmed in the build output.

### Memoisation

| Technique | Applied to |
|---|---|
| `React.memo` | `MetricCard`, `AlertItem`, `NavItemButton`, `InsightCard`, `ProStrategyCard`, `BudgetStrategyCard`, `SpendingComposition`, `FilterPillButton`, `TransactionRow`, `BudgetTracker` |
| `useMemo` | Portfolio donut conic-gradient stops, `generateInsights` output, `activeFilter` computation, `filtered` transactions, `ActivePage` component reference |
| `useCallback` | All event handlers passed as props (theme toggle, filter, track, dismiss, export) |

### Other

- Zero CSS-in-JS runtime — all theming via CSS Custom Properties (one attribute mutation triggers the full cascade)
- Shimmer skeleton animations use a CSS `@keyframes shimmer` + `background-position` trick (GPU-composited, no layout thrash)
- `Intl.NumberFormat` instances declared outside components to avoid re-construction on each render

---

## SEO & Accessibility

### Semantic HTML Landmarks

| Element | Purpose |
|---|---|
| `<aside>` | Sidebar nav |
| `<header>` | Sticky top bar (in `DashboardLayout`) |
| `<main id="main-content">` | Page content region |
| `<nav aria-label="Main navigation">` | Sidebar nav links |
| `<nav aria-label="Tab navigation">` | Header tab bar |
| `<section aria-label="...">` | Alert panel, insight list, transaction table |
| `<article aria-label="...">` | Individual insight/strategy/metric cards |

### ARIA

- `aria-current="page"` on active sidebar nav item
- `aria-current="true"` on active header tab
- `role="progressbar"` + `aria-valuenow/min/max` on all budget/velocity progress bars
- `role="search"` on search input wrapper
- `role="group"` + `aria-label` on filter pill container
- `aria-live="polite"` on insight grid and alert panel (dynamic content)
- `aria-busy="true"` on shimmer skeletons
- `aria-label` on all icon-only buttons (theme toggle, bell, dismiss, nav items)

### Skip Navigation

`DashboardLayout` renders a skip link as the first focusable element:

```html
<a href="#main-content" class="sr-only focus:not-sr-only">Skip to main content</a>
```

Visible only on keyboard focus (`:focus-visible`), positioned fixed top-left.

### Open Graph / Meta

```html
<title>Proton Finance | Wealth Curator Dashboard</title>
<meta name="description" content="Institutional-grade personal finance dashboard..." />
<meta property="og:title" content="Proton Finance — Wealth Curator" />
<meta property="og:description" content="..." />
<meta property="og:type" content="website" />
<meta name="theme-color" content="#0D1117" />  ← updated dynamically by ThemeContext
```

---

## Trade-offs & Decisions

| Decision | Chosen | Alternative | Rationale |
|---|---|---|---|
| **Data layer** | Mock API (`mockData.ts` + `fetchBudget`) | Plaid / real banking API | Real API requires OAuth, PCI compliance, and backend infra out of scope for this deliverable |
| **Insights** | Rule-based engine (5 rules, `insightsEngine.ts`) | ML model (TensorFlow.js) | Rules are auditable, deterministic, and explainable; ML adds ~400KB bundle weight + training data requirements |
| **Filtering** | Client-side (`useMemo` over transaction array) | Server-side pagination | Acceptable for < 500 transactions; O(n) filter is imperceptible; server pagination adds backend round-trips with no UX benefit |
| **Styling** | CSS Custom Properties + inline `CSSProperties` | CSS-in-JS (styled-components) | Zero runtime overhead; theme switching is a free single attribute mutation; better DevTools inspection |
| **Charts** | Recharts (lazy-loaded) | D3 / Victory | Recharts provides React-native charts with a familiar API; lazy loading prevents it from affecting initial bundle |
| **State** | React Context (theme) + local `useState`/`useFetch` | Redux / Zustand | Context is sufficient — there is no cross-slice shared state that would justify a global store |
| **Platform** | React web (Vite) | React Native Web | Web-first requirements met without the cross-platform overhead |
| **Theme toggle** | Two locations (Header + Sidebar bottom) | Header-only | Sidebar toggle serves users who have the sidebar open; both call the same `toggleTheme()` from ThemeContext |
| **Sidebar always-dark** | Hardcoded colours in component | Themed via tokens | Deliberate premium UX: persistent dark nav creates depth and mirrors Bloomberg Terminal / Robinhood Pro conventions |

---

*Proton Finance © 2026 — Wealth Curator Dashboard*
