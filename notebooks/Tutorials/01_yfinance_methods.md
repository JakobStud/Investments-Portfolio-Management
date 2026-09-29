# yfinance Methods for This Project

**Installed version:** yfinance `1.7.0` · **Python:** 3.13 (`.venv`)

> **Golden rule for this project:** yfinance is only a **data-acquisition layer**.
> Its job is to hand us clean price (and dividend/split) time series. Everything
> quantitative — returns, standard deviation, covariance, correlation, beta,
> the efficient frontier, CAPM/APT/Fama-French — **we implement ourselves** in
> `src/portfolio_lab/`. So this file documents the *few* methods that get data
> in, and deliberately skips the large fundamentals/options/news surface we
> don't need.

---

## 1. Mental model: two entry points

yfinance gives you two ways to pull prices. Learn when to reach for each.

| Entry point | Import form | Returns | Use it when |
|---|---|---|---|
| `yf.download(...)` | function | one DataFrame, **MultiIndex columns** (`Price`, `Ticker`) | pulling a **panel of many tickers** at once → the main tool for portfolio work (notebooks 02+) |
| `yf.Ticker("AAPL")` | object | flat-column DataFrame via `.history()`, plus metadata attributes | pulling **one asset** and/or inspecting its corporate actions & metadata |

There is also `yf.Tickers("AAPL MSFT ...")` — a thin container holding many
`Ticker` objects (`.tickers["AAPL"]`). For this project `yf.download` covers the
multi-asset case more directly, so `yf.Tickers` is optional.

---

## 2. `yf.download()` — the workhorse

The primary function. One call fetches one or many tickers over a date range.

**Signature (1.7.0):**
```python
yf.download(
    tickers,                 # "AAPL" or "AAPL MSFT ^GSPC" or ["AAPL","MSFT"]
    start=None, end=None,    # "YYYY-MM-DD" (end is EXCLUSIVE)
    period=None,             # alternative to start/end: "1y","5y","max", ...
    interval="1d",           # "1d","1wk","1mo"; intraday: "1m","5m","1h" (limited history)
    auto_adjust=True,        # <-- IMPORTANT, see §4
    actions=False,           # add Dividends / Stock Splits columns
    group_by="column",       # "column" (default) or "ticker"
    multi_level_index=True,  # True -> MultiIndex columns; False -> flat
    threads=True,            # parallel download of many tickers
    progress=True,           # the [****100%***] progress bar
    rounding=False, prepost=False, repair=False, keepna=False,
    timeout=10, ignore_tz=None, session=None,
)
```

**Parameters that actually matter for us:**

- **`tickers`** — the *reference* to the assets. A space-separated string or a
  list. Exchange suffixes matter: `EQNR` (US ADR) vs `EQNR.OL` (Oslo Børs);
  indices use a caret: `^GSPC` (S&P 500), `^OMXC25` etc. A wrong/empty result
  is almost always a bad ticker string.
- **`start` / `end`** vs **`period`** — give *either* an explicit window
  (`start="2015-01-01", end="2025-01-01"`) *or* a rolling `period`. Note **`end`
  is exclusive** (a common off-by-one). For reproducible research prefer
  explicit `start`/`end` so results don't drift as time passes.
- **`interval`** — `"1d"` for everything in this project. Use `"1mo"` when you
  want to align with **monthly Fama-French factors** (notebook 12). Intraday
  intervals have short history limits (e.g. `"1m"` ≈ last 7 days) — not needed here.
- **`auto_adjust`** — see §4. Leave it at the default `True` for return/risk work.
- **`multi_level_index`** — `True` (default) returns the two-level columns you
  saw: `('Close','AAPL')`. Set `False` for a single ticker if you want flat columns.
- **`group_by`** — `"column"` groups by field first (`Close|High|...`),
  `"ticker"` groups by symbol first. `"column"` is easier for building a matrix
  of one field across many assets.

**Return shape (default, multi-ticker):**
```
columns = MultiIndex[ (Price, Ticker) ]   # Price ∈ {Close,High,Low,Open,Volume}
index   = DatetimeIndex (dates)
```

**Selecting a field correctly (case-sensitive!):**
```python
px = yf.download(["AAPL","MSFT","^GSPC"], start="2015-01-01", end="2025-01-01")
close = px["Close"]                    # -> DataFrame: one column per ticker
aapl  = px[("Close", "AAPL")]          # -> Series (tuple lookup, robust)
close = px.xs("Close", axis=1, level="Price")   # name-based, clearest
```
`px["close"]` (lowercase) raises `KeyError: 'close'` — labels are `Close`, `High`, …

**This `close` DataFrame — dates × tickers — is the input to nearly every
notebook.** From it *you* compute returns and the risk statistics.

---

## 3. `yf.Ticker` + `.history()` — single asset & inspection

```python
t = yf.Ticker("AAPL")
h = t.history(period="5y", interval="1d")   # or start=/end=
```

**`history()` signature (1.7.0):**
```python
history(period=..., interval="1d", start=None, end=None,
        prepost=False, actions=True, auto_adjust=True,
        back_adjust=False, repair=False, keepna=False,
        rounding=False, timeout=10, raise_errors=False)
```

**Key difference from `download`:** `.history()` returns **flat columns** for the
single ticker — `Open, High, Low, Close, Volume`, plus `Dividends`,
`Stock Splits` (because `actions=True` is the default here). No MultiIndex to
unpack. Convenient for one asset; `download` scales better to many.

---

## 4. The adjustment concept (`auto_adjust`) — read this before computing returns

This is the single most important idea for correctness of your risk numbers.

- **`auto_adjust=True` (default in 1.7.0):** the `Close` column is **already
  adjusted** for dividends *and* splits — i.e. a **total-return** price series.
  There is **no separate `Adj Close`** column in this mode. This is what you
  want, so that a dividend payment or a stock split doesn't show up as a fake
  price jump in your returns.
- **`auto_adjust=False`:** you get raw `Close` **and** a separate `Adj Close`.
  Older tutorials assume this and tell you to "use Adj Close" — but on 1.7.0 the
  default already gives you the adjusted series *as* `Close`. Don't double-adjust.

**Practical rule:** keep the default `auto_adjust=True`, and compute returns from
`Close`. If you ever set `auto_adjust=False`, switch to `Adj Close`.

**Corporate-action series (for understanding / total-return checks):**
```python
t.dividends      # Series of dividend payments (dates -> amount)
t.splits         # Series of split ratios
t.actions        # DataFrame combining Dividends + Stock Splits
```
You won't usually need these once you rely on adjusted `Close`, but they explain
*why* the adjustment exists.

---

## 5. Metadata & sanity-check helpers (lightweight)

Cheap calls worth using to validate a download before trusting it:

```python
t.history_metadata     # dict: currency, exchangeName, timezone, instrumentType, ...
t.fast_info            # fast dict-like: last_price, currency, market_cap, ...
t.isin                 # the ISIN identifier
```

Use `history_metadata["currency"]` to catch the classic mistake of mixing assets
quoted in different currencies (e.g. USD vs NOK) before building a covariance
matrix. `fast_info` is the *lightweight* alternative to the heavy `.info` dict.

> `t.info` / `t.get_info()` exists (a large fundamentals dictionary) but is heavy
> and occasionally unreliable — we don't need it for a price/risk project.

---

## 6. What we deliberately DON'T use from yfinance

To keep scope tight: these exist on the `Ticker` object but are **out of scope**
because the project is about prices/risk, and the math is ours:

`balance_sheet`, `income_stmt`, `cash_flow`, `financials`, `earnings*`,
`option_chain` / `options`, `news`, `recommendations`, `analyst_price_targets`,
`institutional_holders`, `sustainability`, `valuation`, `insider_*`.

The **risk-free rate** and **factor returns** do **not** come from yfinance —
they come from **FRED** (`pandas_datareader` / `fredapi`) and the **Kenneth
French Data Library**, documented separately.

---

## 7. Notebook → yfinance call map

| Notebook | What yfinance provides | Typical call |
|---|---|---|
| 01 returns & risk metrics | one asset's adjusted close | `yf.download("AAPL", start=..., end=...)["Close"]` |
| 02 covariance / correlation | panel of N assets' adjusted close | `yf.download([...], start=..., end=...)["Close"]` |
| 03 risk contribution | same panel | same |
| 04 opportunity set / 05 efficient frontier | panel of N assets | same |
| 06 risk-free + optimal risky portfolio | risky assets (rf ← **FRED**) | `yf.download([...])["Close"]` |
| 07 single-index / beta, 08 systematic vs idiosyncratic | assets **+ market index** | `yf.download([..., "^GSPC"])["Close"]` |
| 09 risk pooling / time diversification | long histories | `yf.download([...], period="max")["Close"]` |
| 10 CAPM | assets + market (rf ← **FRED**) | `yf.download([..., "^GSPC"])["Close"]` |
| 11 APT, 12 Fama-French | assets, **monthly** to match factors | `yf.download([...], interval="1mo")["Close"]` |

In every row the deliverable from yfinance is the same object: a **dates × tickers
matrix of adjusted `Close` prices**. Your functions in `src/portfolio_lab/` turn
that matrix into returns and every statistic from there.

---

## 8. Gotchas checklist

- [ ] **Case-sensitive columns** — it's `Close`, not `close`.
- [ ] **MultiIndex columns** from `download` — select with `["Close"]`,
      `[("Close","AAPL")]`, or `.xs(...)`, never `df.Close`.
- [ ] **`end` is exclusive** — add a day if you need the final date included.
- [ ] **`auto_adjust=True` by default** — `Close` is already total-return; there's
      no `Adj Close` in that mode. Don't re-adjust.
- [ ] **Ticker suffixes** — `.OL` (Oslo), `^` (index). Empty result ⇒ check the string.
- [ ] **Mixed currencies** — verify with `history_metadata["currency"]` before
      combining assets into one covariance matrix.
- [ ] **Reproducibility** — prefer explicit `start`/`end` over `period` so a rerun
      next month returns the same window; cache raw pulls into `data/raw/`.
