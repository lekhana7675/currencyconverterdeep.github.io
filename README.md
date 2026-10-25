# Currency converter with rate history

A small web app that converts between currencies and charts how the exchange rate has moved over time. It is built with React, TypeScript and Vite, and organised with [Feature-Sliced Design](https://feature-sliced.design) (FSD).

Rates come from [Frankfurter](https://frankfurter.dev), a free API that republishes European Central Bank (ECB) reference rates. No API key or account is needed.

## Features

- Convert an amount between any two supported currencies using the latest published rate.
- Swap the two currencies with one click. Picking the currency that is already on the other side also swaps them.
- Rate history chart for 1 month, 3 months, 6 months, 1 year or 5 years, with a hover and keyboard tooltip (use the left and right arrow keys).
- Summary of the selected range: change, highest, lowest and average.
- Remembers your amount, currency pair and chosen range between visits (`localStorage`).
- Loading placeholders, error messages with a retry button, and input validation.
- Keyboard accessible, responsive down to phone widths, and respects reduced-motion settings.

## Getting started

Requires Node.js 18 or newer.

```bash
npm install
npm run dev        # start the dev server at http://localhost:5173
```

| Script               | What it does                               |
| -------------------- | ------------------------------------------ |
| `npm run dev`        | Start the Vite dev server                  |
| `npm run build`      | Type-check, then build to `dist/`          |
| `npm run preview`    | Serve the production build locally         |
| `npm run typecheck`  | Run the TypeScript compiler without output |
| `npm test`           | Run unit tests once with Vitest            |
| `npm run test:watch` | Run tests in watch mode                    |

## Project structure (FSD)

FSD splits code into **layers**, then into **slices** (one per business concern), then into **segments** (`ui`, `model`, `api`, `lib`, `config`). Layers sit in a strict order, and code may only import from layers **below** it:

```
app  →  pages  →  widgets  →  features  →  entities  →  shared
```

Two more rules apply:

1. Slices on the same layer must not import each other. (`features/convert-currency` cannot import `features/swap-currencies`. A higher layer composes them instead.)
2. Everything outside a slice imports it through its `index.ts` (its public API), never through its internal files.

```
src/
├── app/                          Entry point, global styles, top-level component
│   ├── main.tsx
│   ├── App.tsx
│   └── styles/index.css          Design tokens (colours, fonts) and base styles
│
├── pages/
│   └── converter/                The only page: header, two widgets, footer
│
├── widgets/                      Self-contained blocks that compose features
│   ├── converter-panel/          Form + swap button + result
│   └── rate-history-panel/       Range tabs + chart + stats
│
├── features/                     User actions that deliver value
│   ├── convert-currency/         Amount form, result display, converter state, parsing
│   ├── swap-currencies/          The swap button
│   └── select-history-range/     Range definitions and range tabs
│
├── entities/                     Business concepts
│   ├── currency/                 Currency list (api), select control (ui)
│   └── rate/                     Latest/history requests, stats, chart, stats panel
│
└── shared/                       Reusable code with no business knowledge
    ├── api/                      Small fetch wrapper
    ├── config/                   API URL, defaults, storage keys
    ├── lib/                      Formatting, dates, useAsync, usePersistedState
    └── ui/                       ErrorNotice, Skeleton
```

### How the pieces connect

- `pages/converter` owns the page layout and calls `useConverterState()` once. It passes the amount and pair to both widgets, so the converter and the history chart always agree.
- `widgets/converter-panel` joins `ConvertForm`, `SwapButton` and `ConversionResult`. It supplies the swap button to the form as a prop, because the two features cannot import each other.
- `widgets/rate-history-panel` joins `RangeTabs`, `RateChart` and `RateStats` and fetches the history for the pair and range.
- `entities/rate` knows how to fetch, summarise and draw rates. It knows nothing about the page that uses them.

### Adding to the app

- **A new range** (for example 2 years): add an entry to `HISTORY_RANGES` in `features/select-history-range/model/ranges.ts`.
- **A new feature** (for example favourite pairs): create `features/favorite-pairs/` with `ui/`, `model/` and an `index.ts`, then render it from a widget or the page.
- **A different rate provider**: change `shared/config` and the two functions in `entities/rate/api/rate-api.ts` plus `entities/currency/api/currency-api.ts`. Nothing else needs to change.

### Optional: enforce the rules automatically

[Steiger](https://github.com/feature-sliced/steiger) is the official FSD linter and flags layer and public-API violations:

```bash
npm install --save-dev steiger @feature-sliced/steiger-plugin
npx steiger ./src
```

## Data source and limits

| Endpoint (base `https://api.frankfurter.dev/v1`) | Used for                |
| ------------------------------------------------ | ----------------------- |
| `GET /currencies`                                | Currency list           |
| `GET /latest?base=USD&symbols=EUR`               | Latest rate for a pair  |
| `GET /{start}..{end}?base=USD&symbols=EUR`       | Rate history for a pair |

Things to know:

- Rates are ECB reference rates, published on working days. There are no weekend values, and the currency list is limited to what the ECB publishes.
- Rates are for information only. They are not a quote for trading or money transfers.
- For longer ranges the API returns weekly rather than daily points, so highest and lowest values reflect that sample. The app shows a note when this applies.
- If you deploy this app, check Frankfurter's current terms and fair-use guidance for your expected traffic.

## Testing

Unit tests cover the pure logic: rate statistics (`entities/rate/lib/stats.test.ts`) and amount parsing (`features/convert-currency/lib/parse-amount.test.ts`).

```bash
npm test
```

## Troubleshooting

| Problem                                | Likely cause and fix                                                                    |
| -------------------------------------- | --------------------------------------------------------------------------------------- |
| "Network error" in the converter       | The browser cannot reach `api.frankfurter.dev`. Check your connection or any ad blocker. |
| Chart says no rates for the period     | The API had no data for that pair and range. Try a longer range or another pair.         |
| Old pair or amount keeps appearing     | Values are stored in `localStorage`. Clear the `fx-converter:*` keys to reset.           |
| Fonts look different                   | Google Fonts did not load. The app falls back to system fonts.                           |

## Tech stack

React 18, TypeScript (strict), Vite, CSS Modules, Vitest. The chart is hand-drawn SVG, so there is no charting dependency.
