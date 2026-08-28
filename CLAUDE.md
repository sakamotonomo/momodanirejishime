# CLAUDE.md

Guidance for Claude Code (and other AI assistants) working in this repository.

## What this is

A single-page web app for **桃谷本店 (Momodani main store)** that turns the nightly
cash-register close (レジ締め) into a printable Japanese business daily report
(営業日報). Staff type the numbers off the register report and the counted cash;
the app derives every total and renders an A4 print sheet that matches the paper
form the head office expects.

The entire application is **one file**: `index.html` (~980 lines). There is no
build step, no package manager, no test suite, and no CI.

```
.
└── index.html    # markup + React/JSX + CSS + all business logic
```

## Running and previewing

There is nothing to install or compile.

```bash
# Simplest: open the file in a browser
xdg-open index.html          # or: open index.html (macOS)

# Preferred while developing (real origin, so localStorage persists per-host)
python3 -m http.server 8000  # then visit http://localhost:8000/
```

**Requires network access on first load.** React 18, ReactDOM 18 and
`@babel/standalone` are pulled from unpkg CDN (`index.html:15-17`). The JSX in
`<script type="text/babel" data-presets="react">` (`index.html:22`) is transpiled
**in the browser at runtime**. Offline, the page stops at the
"読み込んでいます…" boot message.

### Verifying a change

There are no automated tests. Verify by hand in a browser:

1. Load the page, confirm no console errors (a JSX/syntax mistake shows up as a
   Babel error in the console, and the app never mounts).
2. Fill the 入力 (input) tab with realistic numbers and check the 集計 summary.
3. Switch to the 日報 tab and use browser print preview (Ctrl/Cmd-P) — the sheet
   must stay on **one A4 portrait page** with the two-column layout intact.
4. Press 保存する, reload the page, and reopen the date from the
   保存した日を開く dropdown to confirm round-tripping through localStorage.

## Architecture

Everything lives inside the single `<script type="text/babel">` block.

| Region | Lines | Contents |
| --- | --- | --- |
| `store` | 24-36 | localStorage wrapper with a promise-based `list/set/get/delete` API |
| Constants | 45-55 | `ROLLS`, `POINT_UNIT`, `DEFAULT_FUND` |
| `blank()` | 64-88 | the single source of truth for form state shape |
| Helpers | 90-114 | `n`, `yen`, `arr3`, `sum3`, `jpDate`, `lastYearISO` |
| `App()` | 116-789 | state, the `c` calculation memo, both tabs' JSX |
| `c = useMemo` | 140-217 | **all derived numbers** — the business core |
| Persistence | 203-248 | `loadIndex`, `save`, `load`, `reset` |
| Input tab JSX | 277-528 | red-bordered `.card` sections = operator input |
| Report tab JSX | 530-789 | `.sheet` — the printable 営業日報 |
| Small components | 791-824 | `cellStaff`, `Field`, `Money`, `Row` |
| `CSS` | 826-974 | the whole stylesheet as a template literal |

State is a single flat object `f` (from `blank()`), held in one `useState`.
Every derived value is computed in the `c` memo and read from there — **no
derived value is ever stored in state**. Keep it that way: if you need a new
number in the report, add it to `c`, not to `f`.

### The calculation pipeline (`index.html:140-217`)

Domain rules are encoded here; change them only with a clear business reason.

- `rollTotal` — coin rolls (棒金) valued at fixed yen per roll: 500円→25,000,
  100円→5,000, 50円→2,500, 10円→500, 5円→250, 1円→50 (`ROLLS`, line 45).
- `cashless` = credit + ICOCA + PayPay + offline.
- `gift` = point cards × `POINT_UNIT` (300 yen per card).
- `counted` = bills + coins + shop receipts + rolls + cashless + gift − `fund`
  (the 300,000 yen change float, `DEFAULT_FUND`).
- `diff` = `counted` − `receiptTotal`; split into `plus`/`minus` for the form.
  `|diff| <= 1000` renders the green "許容範囲" box, otherwise the red recount
  warning (`diffOK`, line 251).
- `sabiki1` (差引①) = `receipt + diff` — algebraically equal to `counted`, but
  written this way deliberately to mirror the paper form's
  レジ合計 ＋プラス −マイナス row. Do not "simplify" it away.
- `expense` (支出合計②) = shop receipts + cashless + gift;
  `cash` = ①−②; `toHQ` (差引本社入金額) = cash − transfer.
- **Tax split**: bag sales (袋売上) are 10% goods, everything else is 8% food.
  `bagIncl` is clamped into `[0, totalIncl]`. Tax-exclusive amounts use
  `Math.floor(incl / 1.08)` and `Math.floor(incl / 1.1)`; the tax line is the
  remainder, so A + tax8 + B + tax10 always reconciles exactly to the
  tax-inclusive total. `net` = A + B is the headline 本日の売上（税抜）.
- Year-on-year ratios (`salesRatio`, `guestRatio`, `cumRatio`) are `null` when
  the prior-year denominator is 0, and render as an empty cell rather than
  `NaN%` or `∞`. Preserve that null-means-blank convention for any new ratio.

### Storage

`localStorage`, one entry per business day:

```
key:   nippo:YYYY-MM-DD
value: JSON.stringify(f)      // the whole form object
```

`store.list("nippo:")` builds the date dropdown. Data never leaves the device —
there is no server, no fetch, no analytics. **Do not add network calls that
transmit report data**; the UI explicitly promises local-only storage
(`index.html:525-527`).

`load()` (line 226) re-hydrates defensively: it spreads over `blank()`, merges
`rolls`, and runs `shopReceipt`/`coins` through `arr3`. `arr3` (line 97) exists
because older saved records stored those fields as a single scalar rather than a
3-element array. **Every schema change must keep old saved records loadable** —
follow the same merge-over-defaults pattern instead of assuming a field's shape.

## Conventions

- **Language.** All UI strings, labels and code comments are Japanese. Match
  that; do not translate existing text to English.
- **Dates.** ISO `YYYY-MM-DD` in state and storage. `jpDate()` converts to the
  Reiwa era for display with `r = year - 2018` (line 104-109) — hardcoded, and
  correct only for Reiwa.
- **Numbers.** All form fields are **strings** (including numeric inputs). Never
  do arithmetic on `f.*` directly — always pass through `n()` (line 90), which
  strips non-numeric characters and returns 0 for anything unparseable. Format
  for display with `yen()` (line 94, `toLocaleString("ja-JP")`).
- **Styling.** One template literal `CSS` (line 826) injected via
  `<style>{CSS}</style>` (line 255). Colors come from CSS custom properties on
  `.app` (`--ink`, `--paper`, `--red`, `--gold`, `--ok`, …). No inline styles
  except one-off spacing; no CSS framework.
- **Print.** `.no-print` hides chrome; `@page{size:A4 portrait;margin:10mm}` and
  the print media query live at the end of `CSS`. The report body uses a Mincho
  serif stack to match the paper form, while numeric cells switch back to a
  gothic sans for tabular figures.
- **Input affordance.** A red left border (`.card.red`) and red-outlined
  `Money` fields mark "the operator must type this here". Purely derived
  sections stay neutral. Keep that visual contract when adding fields.
- **Components.** Use the existing `Field`, `Money` (label + right-aligned
  numeric input + unit) and `Row` (summary line) rather than hand-rolling markup.
- **Staff grid.** The report renders a fixed 12-row × 2-column staff table
  (24 slots) regardless of how many entries exist; `cellStaff` fills empty slots
  with `（　　〜　　）`. Extra staff beyond 24 will not print.

## Adding a form field — checklist

1. Add the key with an empty-string default to `blank()` (line 64).
2. Render an input in the appropriate 入力 `<section>` using `Money`/`Field`
   with the `set("key")` handler.
3. If it feeds a total, derive it inside the `c` memo (line 140) — not in state.
4. Add the corresponding cell to the 日報 sheet JSX and check print layout.
5. Confirm a record saved before your change still loads (see `load()` above).

## Gotchas

- Runtime Babel transpilation means **a syntax error is silent** except in the
  console — the page just sits on the boot text. Always check the console.
- No bundler: `import`/`export`, npm packages, TypeScript and JSX-in-separate-
  file are all unavailable. Keep everything in `index.html` and in browser-
  runnable JSX unless the user explicitly asks to introduce tooling.
- The CDN scripts are unpinned to a patch version (`react@18`). If the app
  suddenly breaks with no local change, suspect the CDN.
- `Math.floor` on the tax split means rounding is deliberately *toward the
  store's favor on tax*; don't switch to `Math.round` without asking.

## Git workflow

- Default branch: `main`. Development for this task happens on
  `claude/claude-md-docs-8pgxwz`.
- Push with `git push -u origin <branch-name>`.
- Do not open a pull request unless explicitly asked.
