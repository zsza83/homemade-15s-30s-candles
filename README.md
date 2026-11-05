# homemade-15s-30s-candles
Homemade 15s/30s OHLCV from WS last — micro-exit &amp; stagnation notes. Open questions welcome.
# Dockit: 15s / 30s Candle Usage (BUKABOT)

**Purpose:** Clear description of our homemade 15s/30s OHLCV roles — for GitHub / peer review.  
**Important:** **Entries are not candle-driven** (Binance→Bybit lead-lag: tick gap + slip + depth + leader Δ%). 15s/30s = **micro-exit / stagnation / (optional) patterns**, not an entry trigger.

---

## 1. Why 15s / 30s at all?

Public exchange OHLCV usually starts at **1m**. We build buckets from WS **last price** ticks:

| TF | Role (current live code) |
|---|---|
| **15s** | Active-trade **exit** (closes against entry; post–break-even stagnation if stop idle) |
| **30s** | During search: **stagnation strikes** (Binance 30s move ≈ 0 → “dead” coin counter; swap-motor hook) |

This is **not** the usual cross-exchange arb README pattern (tick / mid / L2). Intent: short-horizon **exit / cleanup** while the gap strategy decides on ticks.

---

## 2. Data path — end to end

```
Binance / Bybit public WS (ticker)
        │
        ▼
TickProcessor._ingest_ticker_price()
  · live_prices[EXCH_SYMBOL] = price
  · Binance: tick ring (leader momentum)     ← for entry
  · candle_engine.update_candle_data_with_exchange(EXCH, symbol, price)
        │
        ▼
CandleEngine (src/modules/candle_engine.py)
  · bucket_15s = floor(now/15)*15
  · bucket_30s = floor(now/30)*30
  · open candle: current_15s_candle / current_30s_candle
  · on bucket flip: closed OHLCV → candles_history[KEY], max 16 bars
        │
        ├── 15s → TradeManager exit (open position)
        └── 30s → main._update_stagnation_strikes (search, throttled)
```

### Key format

```
{EXCHANGE}_{TF}_{SYMBOL}
e.g. BYBIT_15S_SOLUSDT
     BINANCE_15S_SOLUSDT
     BINANCE_30S_NEARUSDT
```

`SYMBOL` = unified `BASEUSDT` (no dashes/slashes).

### Closed candle fields

```python
{
  "timestamp": <bucket_start_unix>,  # start of 15s/30s window
  "open":  float,
  "high":  float,
  "low":   float,
  "close": float,
}
```

Close time in logic is often: `timestamp + 15` (for 15s TF).

### Buffer size

`append_and_truncate_candle_history(..., max_limit=16)`  
→ ~16 × 15s ≈ **4 minutes** on 15s; ~16 × 30s ≈ **8 minutes** on 30s (rolling window).

### CPU during a live trade

While a trade / open / close is critical, tick ingest **only feeds candles for the active coin** (other symbols skipped) so tracking does not lose milliseconds.

---

## 3. Where the code lives

| File | Responsibility |
|---|---|
| `src/modules/candle_engine.py` | 15s/30s bucket build, history |
| `src/modules/candle_logic/candle_manager.py` | truncate + movement % |
| `src/modules/candle_logic/candle_downloader.py` | `calculate_candle_movement_pct`, (REST 1m hist — not hot path) |
| `src/modules/candle_logic/pattern_detector.py` | patterns (PUMP / EXHAUSTION) — **not wired into live entry/exit** |
| `src/modules/bot_normalizer.py` | `setup_candle_engine`, snapshot load/save |
| `src/modules/tick_processor.py` | WS → ingest → candle update |
| `src/modules/trade_manager.py` | **15s exit** |
| `src/main.py` | **30s stagnation strikes** (every Nth fan-out pass) |
| `config/settings.yaml` | `exit_15s_*` |

---

## 4. 15s — micro-exit detail

Called from `TradeManager.execute_tick_tracking()` on every trade tick (main loop trade branch).

### 4.1 Preconditions

- `active_trade_ticker` set, valid `entry_price`
- Not `_entry_provisional` / `_is_opening_now` (no exit during order RTT)
- Price not absurd (`ratio` in 0.5…1.5)

### 4.2 Tick-level cuts (no candles)

1. **Momentum shield:** before BE, if profit &lt; −0.10% and 3 negative ticks → immediate close  
2. **Break-even (BE):** profit ≥ +0.15% → `last_milestone = 1`, stop = entry  
3. **Trailing:** after BE, VPS trailing; trigger → close  

### 4.3 15s trend-exit (after BE)

Config: `exit_15s_trend_closes` (default **2**).

```
_be_trend_exit_signal(trend_need):
  · last N closed 15s candles (Bybit_15S, fallback Binance_15S)
  · every close is AGAINST entry:
      LONG  → close < entry
      SHORT → close > entry
  · candles closed after BE timestamp
  → True = [15S TREND EXIT] market close
```

**Meaning:** after BE, if the short TF keeps closing below entry (long), do not hold on hope.

### 4.4 15s stagnation-exit (after BE)

Config: `exit_15s_stagnation_candles` (default **3**).

```
After BE:
  if ≥ N 15s candles closed since last stop update
  AND stop did not move → [AUTO EXIT] timeout-style close

Before BE:
  not candle-based — 60s wall clock with no stop move
```

Absolute timeout: **200 s** from entry (independent of candles).

### 4.5 Where 15s history comes from

```python
_get_15s_candle_history():
  BYBIT_15S_{active}  preferred
  else BINANCE_15S_{active}
```

Bybit is the execution venue — hence Bybit 15s first.

---

## 5. 30s — search stagnation (not exit)

`main._update_stagnation_strikes()` — during fan-out, every `stagnation_every_n_scans` passes (default 5).

```
BINANCE_30S_{coin} last 3 candles
movement_pct = |close_last - open_first| / open_first * 100
if movement ≤ 0.0001 → stagnation_strikes[coin] += 1
else clear strike
```

**Today:** counter / clear only. The old “single swap after 16 strikes” path is thinned out of `main`; coin replace is mostly shitcoin/quarantine `rolling_swap` + background **universe research** (900s).

---

## 6. What candles deliberately do **not** do

| Not used for | Instead |
|---|---|
| Entry signal | Gross/net gap, `expected_slip` / `slip_by_coin`, L5 depth, Binance leader Δ% / 1s |
| RSI / 15m EMA confluence | `rsi_trigger` exists in config; open gate does not use it |
| Two-leg arb timing | Bybit spot long only (Binance = leader signal) |
| Official exchange 15s OHLCV | None — homemade aggregation from last |

---

## 7. Related but separate: `pattern_detector`

`detect_candle_patterns()`:

- `PARABOLIC_PUMP` — 3 growing green bodies  
- `INSTITUTIONAL_EXHAUSTION` — large upper wick  

**Status:** module exists; live entry/exit path **does not call it**. Review question: wire into 15s/30s exit or skip filters, or leave as research-only.

---

## 8. Live config

```yaml
# config/settings.yaml — strategy
exit_15s_stagnation_candles: 3   # after BE: N×15s with no stop move → exit
exit_15s_trend_closes: 2         # after BE: N×15s closes against entry → exit
stagnation_every_n_scans: 5      # throttle 30s strike updates
```

Hardcoded in tracking (not yaml): BE threshold +0.15%, momentum −0.10%×3 ticks, absolute 200s, pre-BE 60s stagnation.

---

## 9. Memory / lifecycle

- Max 16 candles per history key  
- `process_live_price_and_candles`: if `candles_history` &gt; 500 keys → flush (safety)  
- Swap/purge may drop 15s/30s keys for removed coins  
- Snapshot: disk save / cold-boot restore (snapshot fresher than 5 minutes)

---

## 10. Open questions for review

Left open on purpose — “why not this way?” comments:

1. **Entry:** stay tick-only, or soft filter: last closed Binance 15s green?  
2. **Trend-exit N=2 / 15s:** too fast / too slow vs EU spot round-trip fees?  
3. **Bybit vs Binance 15s for exit:** execution venue correct, or should leader (Binance) close decide?  
4. **30s stagnation:** strikes → real satellite swap, or is universe refresh every 900s enough?  
5. **pattern_detector:** wire into exit/skip, or delete noise?  
6. **Bucket align:** `floor(now/15)*15` vs exchange server time — drift risk?  
7. **16-bar window:** enough for trend-exit, or longer history needed?  
8. On serious stacks (Rust ingest), is a 15s exit TF common at all, or tick stops only?

---

## 11. One-line summary

> 15s/30s candles are a **homemade micro-TF from WS last**: **15s = post-BE micro-exit (trend + stagnation)**, **30s = search-time dead-coin signal**; **entry** remains **tick lead-lag + fee/slip/depth**, not a candle signal.

---

*This doc matches the live path (`main.py` fan-out + `trade_manager` tracking). Older `r-*` / backup mains may have richer 30s swap logic — that is not the default path.*

---

## Related: own-stack market data (sister dockit)

15s/30s answer *how* we bucket WS last. Separately: *where* durable history lives when JSON is too slow on the hot path.

- Live article: https://vpnstronger.com/own-stack-market-data-for-daytrade-bots-json-is-too-slow-what-instead
- Repo + open questions: https://github.com/zsza83/own-stack-market-data

Same stance: **94 is not 100**. Not financial advice.

---


## Related: own activity calendar (sister OwnStack dockit)

Same Own Development Log pattern — familiar shape, own implementation, zero external widget.

- Live article: https://vpnstronger.com/own-activity-calendar-familiar-shape-zero-external-widget
- Repo + open questions: https://github.com/zsza83/own-activity-calendar
- Homepage Activity block: https://vpnstronger.com

Not trading. Same sovereignty stance: no borrowed contribution embed for a pretty graph.
## Related idea: shock-fade (not the gap path)

Same stack, different job. The sections above are **gap lead-lag**: entries from ticks; 15s/30s only for micro-exit / dead-coin cleanup.

There is a **parallel paper path** (`leader_momentum`, regime `shock_2pct`) that *does* lean on higher TF bars for the **entry gate** — still homemade / own OHLCV, not “candle patterns as strategy”:

| Piece | Role |
| ----- | ---- |
| **BTC 15m** | Shock gate: \|Δ\| ≥ ~2% → risk-on window (`WAIT_SHOCK` until then) |
| **Alt 5m** | Fade setup: dump ≥ ~0.60% → candidate **LONG** (mean-revert after shock, not continuation) |
| **Exit** | Bybit quote only + short grace after open (no mixed-venue mid for exit) |
| **Fees** | EU Bybit taker RT ~**0.50%** — only rare shocks clear the bar; calm CTRL is underwater |

### Why it belongs next to 15s/30s

- Still: **no official exchange 15s OHLCV**; buckets / history from own ingest.
- Still: **not** RSI/EMA confluence theatre — gate is regime + dump size vs fee floor.
- Contrast with §1–§6: gap bot keeps entry tick-only; shock-fade *allows* TF bars as a **filter**, then tick/venue discipline on exit.

### Working notes (paper, not live claim)

1. Shock-only (`lm_trade_regimes_only: [shock_2pct]`) — CTRL off on purpose.
2. Cluster / walk-forward can look fine **before** fees; after 0.50% RT the edge is sparse → acceptance must be fee-honest.
3. Open questions same spirit as §10: Is 15m BTC the right shock TF? Fade vs chase? Hold timeout vs 15s micro-exit reuse?

### One-line

> Homemade candles are not only micro-exit: the same own-stack idea can gate a **rare BTC-shock → alt-fade LONG**, but only if fees leave room — **94 is not 100**.

Not financial advice. Open questions welcome on the issues / discussions of this repo.
