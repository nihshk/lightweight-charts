# Project: Custom Financial Scripting Language (DSL) & Lightweight Charts Engine

## 1. Executive Summary & Objective
Build a complete, high-performance Domain-Specific Language (DSL) inspired by Pine Script, paired with an interactive financial charting terminal powered by `lightweight-charts`. Users write custom indicator and strategy scripts in an integrated code editor; the compiler tokenizes, parses, and evaluates the script against historical and real-time OHLCV bar arrays, rendering the output directly onto the HTML5 canvas chart.

---

## 2. Tech Stack & Tooling
- **Frontend Framework:** React 18+ with TypeScript (Vite bundler)
- **Charting Engine:** `lightweight-charts` (v4.x)
- **Code Editor:** `@monaco-editor/react` or CodeMirror 6 (with custom syntax highlighting for our DSL)
- **Styling:** Tailwind CSS
- **Testing:** Vitest (strictly required for Lexer, Parser, and Interpreter unit tests)

---

## 3. Commands & Workflow
- `npm install` - Install dependencies
- `npm run dev` - Start local Vite dev server
- `npm test` - Run Vitest suite (Always run tests after modifying `/src/compiler/`)
- `npm run build` - Type-check (`tsc`) and build production bundle

---

## 4. Directory Architecture
```text
src/
├── compiler/
│   ├── types.ts          # Token types, AST node interfaces, Runtime Value types
│   ├── lexer.ts          # Hand-written scanner converting raw source string -> Token[]
│   ├── parser.ts         # Recursive descent parser converting Token[] -> AST (ProgramNode)
│   ├── stdlib.ts         # Built-in TA functions (sma, ema, rsi, macd, crossover, highest, lowest)
│   ├── evaluator.ts      # Vectorized/Bar-by-bar AST runtime executing against OHLCV data
│   └── errors.ts         # Custom SyntaxError and RuntimeError classes with line/column tracking
├── chart/
│   ├── ChartContainer.tsx # Main Lightweight Charts wrapper managing chart instance & resize
│   ├── SeriesManager.ts   # Synchronizes compiler PlotOutput[] with chart Line/Histogram/Candlestick series
│   └── primitives/        # Custom Lightweight Charts ISeriesPrimitive plugins (backgrounds, shapes)
├── components/
│   ├── ScriptEditor.tsx   # Code editor with "Compile & Run" action and error console
│   ├── TimeframeBar.tsx   # Symbol and timeframe (1m, 5m, 15m, 1H, 1D) selector
│   └── StrategyStats.tsx  # Displays PnL, Win Rate, Max Drawdown when strategy() calls are used
├── data/
│   └── sampleOhlcv.ts     # Mock historical OHLCV bar generator & WebSocket simulator
└── App.tsx                # Main layout splitting Chart, Editor, and Output Console
```

---

## 5. Custom Language Specification (Grammar & Semantics)

### 5.1 Data Model: Series-First Architecture
Every variable in the language is implicitly a **Time Series** (an array aligned to the OHLCV bar index `0..N-1`) or a **Scalar** (broadcasted across all bars).
- **Built-in Series Variables:** `open`, `high`, `low`, `close`, `volume`, `time`, `hl2`, `hlc3`, `ohlc4`, `bar_index`
- **Historical Lookback Operator (`[]`):** Accessing prior bar values.
  - Example: `close[1]` returns the previous bar's close (`NaN` at index 0).
  - Example: `sma(close, 14)[2]` returns the SMA value from 2 bars ago.

### 5.2 Lexical Tokens
- **Keywords:** `var`, `if`, `else`, `true`, `false`, `and`, `or`, `not`, `na`
- **Literals:** `NUMBER` (float/int), `STRING` (single or double quotes), `COLOR` (`#RRGGBB` or `#RRGGBBAA`)
- **Operators:**
  - Arithmetic: `+`, `-`, `*`, `/`, `%`
  - Comparison: `>`, `<`, `>=`, `<=`, `==`, `!=`
  - Ternary: `? :`
  - Assignment: `=` (variable declaration/assignment), `:=` (reassignment)
- **Delimiters:** `(`, `)`, `[`, `]`, `,`
- **Comments:** `//` single-line comments (ignored by Lexer)

### 5.3 Formal EBNF Grammar (Recursive Descent Precedence)
Implement the parser following this exact operator precedence (lowest to highest):
1. `Program`            -> `Statement* EOF`
2. `Statement`          -> `AssignmentStmt | ExpressionStmt`
3. `AssignmentStmt`     -> `IDENTIFIER ("=" | ":=") Expression`
4. `Expression`         -> `TernaryExpr`
5. `TernaryExpr`        -> `LogicalOrExpr ("?" Expression ":" Expression)?`
6. `LogicalOrExpr`      -> `LogicalAndExpr ("or" LogicalAndExpr)*`
7. `LogicalAndExpr`     -> `EqualityExpr ("and" EqualityExpr)*`
8. `EqualityExpr`       -> `ComparisonExpr (("==" | "!=") ComparisonExpr)*`
9. `ComparisonExpr`     -> `AdditiveExpr ((">" | "<" | ">=" | "<=") AdditiveExpr)*`
10. `AdditiveExpr`      -> `MultiplicativeExpr (("+" | "-") MultiplicativeExpr)*`
11. `MultiplicativeExpr` -> `UnaryExpr (("*" | "/" | "%") UnaryExpr)*`
12. `UnaryExpr`         -> `("-" | "not") UnaryExpr | PostfixExpr`
13. `PostfixExpr`       -> `PrimaryExpr ( "[" Expression "]" | "(" ArgumentList? ")" )*`
14. `ArgumentList`      -> `Argument ("," Argument)*`
15. `Argument`          -> `(IDENTIFIER "=")? Expression` *(Supports both positional and named arguments like `color=#ff0000`)*
16. `PrimaryExpr`       -> `NUMBER | STRING | COLOR | "true" | "false" | "na" | IDENTIFIER | "(" Expression ")"`

---

## 6. Built-In Standard Library (`stdlib.ts`)

### 6.1 Technical Analysis Functions (Vectorized Series Input -> Series Output)
All TA functions must gracefully propagate `NaN` (`na`) during the warmup period.
- `sma(source: Series, length: number): Series` - Simple Moving Average
- `ema(source: Series, length: number): Series` - Exponential Moving Average (alpha = 2 / (length + 1))
- `rsi(source: Series, length: number): Series` - Wilder's Relative Strength Index
- `stdev(source: Series, length: number): Series` - Rolling Standard Deviation
- `highest(source: Series, length: number): Series` - Rolling maximum over `length` bars
- `lowest(source: Series, length: number): Series` - Rolling minimum over `length` bars
- `crossover(seriesA: Series, seriesB: Series): Series<boolean>` - True on bar i if A[i] > B[i] and A[i-1] <= B[i-1]
- `crossunder(seriesA: Series, seriesB: Series): Series<boolean>` - True on bar i if A[i] < B[i] and A[i-1] >= B[i-1]
- `nz(source: Series, replacement = 0): Series` - Replaces `NaN` values with `replacement`
- `change(source: Series, length = 1): Series` - Difference between current bar and `length` bars ago

### 6.2 Plotting & Visual Directive Functions
Directive functions do not just return math; they push descriptors to the Evaluator's `RenderQueue`.
- `plot(series, title?, color?, linewidth?, style?, pane?)`
  - Emits a `LineSeries` or `HistogramSeries` descriptor.
  - `pane`: `"main"` (overlay on price) or `"subpane"` (oscillator sub-pane like RSI).
- `hline(price, title?, color?, linestyle?, pane?)`
  - Attaches a horizontal price line (`series.createPriceLine()`) to the active pane.
- `plotshape(conditionSeries, title?, style?, location?, color?, text?)`
  - Converts truthy bar indices into `SeriesMarker` objects (`series.setMarkers()`), sorted strictly in ascending chronological order.
- `bgcolor(conditionSeries, color?)`
  - Highlights vertical background bands where `conditionSeries` is true.

### 6.3 Strategy & Backtesting Directives
- `strategy.entry(id: string, direction: "long" | "short", when: Series<boolean>)`
- `strategy.close(id: string, when: Series<boolean>)`
  - Evaluates simulated trades at the next bar's `open` (or current bar `close`), generating buy/sell chart markers and computing cumulative PnL, Win Rate %, Profit Factor, and Max Drawdown.

---

## 7. Compiler & Evaluator Contract

The compiler pipeline must expose a single pure function for the UI to call:

```typescript
export interface OHLCVBar {
  time: number; // Unix timestamp in seconds
  open: number;
  high: number;
  low: number;
  close: number;
  volume: number;
}

export interface PlotDescriptor {
  id: string;
  type: 'line' | 'histogram';
  title: string;
  color: string;
  lineWidth: number;
  pane: 'main' | 'subpane';
  data: Array<{ time: number; value: number; color?: string }>;
}

export interface ExecutionResult {
  success: boolean;
  plots: PlotDescriptor[];
  markers: Array<{
    time: number;
    position: 'aboveBar' | 'belowBar' | 'inBar';
    color: string;
    shape: 'arrowUp' | 'arrowDown' | 'circle' | 'square';
    text?: string;
  }>;
  priceLines: Array<{ price: number; title: string; color: string; pane: 'main' | 'subpane' }>;
  strategyReport?: {
    totalTrades: number;
    winRate: number;
    netProfit: number;
    maxDrawdown: number;
  };
  errors: Array<{ line: number; column: number; message: string }>;
}

export function compileAndRun(sourceCode: string, bars: OHLCVBar[]): ExecutionResult;
```

---

## 8. Lightweight Charts Integration Rules (`SeriesManager.ts`)
1. **No Full Chart Re-creation on Script Edit:** Keep the base `IChartApi` and main `CandlestickSeries` alive. Maintain a registry map of dynamically created indicator series (`Map<string, ISeriesApi<any>>`) and call `chart.removeSeries()` cleanly before applying new script outputs.
2. **Filter `NaN` Values:** `lightweight-charts` crashes or misrenders if passed `NaN` or `Infinity`. Strip or convert `NaN` data points into whitespace data `{ time }` before passing arrays to `series.setData()`.
3. **Chronological Ordering & Deduplication:** Ensure all `time` values passed to `setData()` and `setMarkers()` are strictly ascending with no duplicate timestamps.
4. **Multi-Pane Support:** For plots with `pane: "subpane"`, either configure separate `priceScaleId` margins (e.g., main chart `top: 0.05, bottom: 0.30`; oscillator scale `top: 0.75, bottom: 0.05`) or synchronize a secondary `IChartApi` instance underneath via `subscribeVisibleLogicalRangeChange`.

---

## 9. Step-by-Step Implementation Roadmap for Claude Code
Execute these phases sequentially. **Do not jump to Phase 4 until Phases 1–3 pass all unit tests.**

### Phase 1: Lexer & Tokenizer
- Implement `/src/compiler/types.ts`, `/src/compiler/errors.ts`, and `/src/compiler/lexer.ts`.
- Handle numbers, strings, hex colors (`#ff0055`), identifiers, operators, brackets, and line/column tracking.
- Write unit tests in `/src/compiler/__tests__/lexer.test.ts` verifying token streams and error reporting.

### Phase 2: Recursive Descent Parser & AST
- Implement `/src/compiler/parser.ts` adhering strictly to Section 5.3 EBNF precedence rules.
- Support positional and named function arguments (`plot(my_rsi, title="RSI", color=#00ff88)`).
- Support postfix array lookback (`close[1]`, `(high - low)[2]`).
- Write unit tests in `/src/compiler/__tests__/parser.test.ts` verifying AST generation.

### Phase 3: Vectorized Evaluator & Standard Library
- Implement `/src/compiler/stdlib.ts` and `/src/compiler/evaluator.ts`.
- Support vectorized arithmetic (`high - low` subtracts element-wise across all bars; `close * 2` broadcasts scalar `2` across `close`).
- Implement `plot()`, `hline()`, `plotshape()`, and `strategy.entry()/close()` side-effects.
- Write unit tests in `/src/compiler/__tests__/evaluator.test.ts` using sample OHLCV arrays.

### Phase 4: Lightweight Charts Bridge & UI Terminal
- Build `ChartContainer.tsx` and `SeriesManager.ts` to render candlesticks, dynamic indicator overlays, sub-pane oscillators, and strategy markers.
- Build `ScriptEditor.tsx` with preset script templates (e.g., "EMA Crossover Strategy", "RSI Overbought/Oversold", "Bollinger Bands").
- Add real-time error highlighting and an output console displaying compilation time and strategy backtest metrics.

---

## 10. Example Target Script (Must Compile & Render Out-of-the-Box)
The completed system must parse, evaluate, and render the following sample script without errors:

```pine
// Dual EMA Crossover + RSI Filter Strategy
fast_len = 9
slow_len = 21

fast_ema = ema(close, fast_len)
slow_ema = ema(close, slow_len)
rsi_val  = rsi(close, 14)

// Plot Moving Averages on Main Chart
plot(fast_ema, title="Fast EMA", color=#00bcd4, linewidth=2)
plot(slow_ema, title="Slow EMA", color=#ff9800, linewidth=2)

// Plot RSI in Subpane
plot(rsi_val, title="RSI 14", color=#ab47bc, linewidth=2, pane="subpane")
hline(70, title="Overbought", color=#ef5350, pane="subpane")
hline(30, title="Oversold", color=#26a69a, pane="subpane")

// Signal Logic
bull_cross = crossover(fast_ema, slow_ema) and rsi_val > 50
bear_cross = crossunder(fast_ema, slow_ema) and rsi_val < 50

plotshape(bull_cross, title="Buy", style="arrowUp", location="belowBar", color=#26a69a, text="BUY")
plotshape(bear_cross, title="Sell", style="arrowDown", location="aboveBar", color=#ef5350, text="SELL")

strategy.entry("Long", "long", when=bull_cross)
strategy.close("Long", when=bear_cross)
```
