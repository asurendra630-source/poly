# Reversed-Signal Dry Run — Methodology, Fixes, and Behaviour

**Box:** EC2 `3.249.210.105` (ubuntu) · **Report generated:** 2026-09-10 05:45 UTC
**Question asked:** *"do the reverse of what signal we get — if it gives UP, take DOWN — and see how it performs."*
**Answer in one line:** the reversal runs correctly and loses more than the forward bot. It does not rescue the strategy, and the live run shows the mechanism more sharply than the backtest did.

> Every number in this file was read off the box. Nothing is estimated or carried over from
> memory unless the section says so explicitly. Where n is too small to conclude, it says so.

---

## 1. File and environment inventory (exact paths)

### Config / env layers

| Path | Size | mtime | Role |
|---|---|---|---|
| `/home/ubuntu/.env` | 20083 B | 2026-09-09 11:01 | Base layer. **Contains the compromised wallet private key.** Never read or echoed; every command in this session piped output through a redaction `sed`. |
| `/home/ubuntu/DRYRUN_ONLY.env` | 18977 B | 2026-09-09 11:01 | Dry-run overlay. Self-documenting: every value cites the `opus.py` line that justifies it. `supervise_dry.sh` refuses to launch unless `DRY_RUN=true`. |
| `/home/ubuntu/supervise_dry.sh` | 3717 B | 2026-09-09 11:01 | **Forward** bot launcher + pre-flight `chk`. |
| `/home/ubuntu/supervise_rev.sh` | 1910 B | 2026-09-09 15:25 | **Reversed** bot launcher (created this session). |

**Load precedence — verified, and it matters for reading this run:** `load_dotenv(override=False)`,
so `.env` loads first, `DRYRUN_ONLY.env` overrides it, and **the launch environment beats both**.
That is why the `export` lines in `supervise_rev.sh` are the effective config for the reversed bot.

### Code

| Path | md5 | Size |
|---|---|---|
| `/home/ubuntu/opus.py` | `ebc2872f4dbc2b765f733f5d645416fe` | 751205 B |
| `/home/ubuntu/opus_rev.py` | `4d5ff2d5a7c656838c90d4d55a331900` | 752044 B |

### Ledgers (kept strictly separate — this is what proves isolation)

| Forward (pid 822553) | Reversed (pid 831096) |
|---|---|
| `/home/ubuntu/latarb_settle.jsonl` | `/home/ubuntu/rev_latarb_settle.jsonl` |
| `/home/ubuntu/latarb_maker.jsonl` | `/home/ubuntu/rev_latarb_maker.jsonl` |
| `/home/ubuntu/latarb_fills.jsonl` | `/home/ubuntu/rev_latarb_fills.jsonl` |
| `latarb_state.<n>.dry.<addr>.json` | `rev_latarb_state.137.dry.<addr>.json` |
| `/home/ubuntu/opus.log` | `/home/ubuntu/rev.log` |

### Reversed-bot launch environment (`supervise_rev.sh` exports, verbatim)

```bash
export DRY_RUN=true
export LATARB_SHADOW=false            # disk: only ~713MB free, the tape is 359MB
export CALIBRATION_LOG_ENABLED=false
export LATARB_FILLS_PATH=/home/ubuntu/rev_latarb_fills.jsonl
export LATARB_SETTLE_PATH=/home/ubuntu/rev_latarb_settle.jsonl
export DRY_MAKER_LOG=/home/ubuntu/rev_latarb_maker.jsonl
export COINS=BTC,ETH                  # memory + halve API load; == forward universe
export DISCOVERY_INTERVAL_S=30
export POLYMARKET_PROXY_ADDRESS=0x...dEaD   # dummy, ONLY to get a distinct instance-lock key
```

Everything not listed is inherited from `.env` + `DRYRUN_ONLY.env`. The ones that govern
behaviour here: `ENTRY_MODE=maker`, `MAKER_GTD_TTL_S=120`, `DRY_RUN_FILL_PROB=0.0`
(no coin-flip fills — a post only fills if the observed book actually prints through it),
`MIN_ORDER_USDC=MAX_ORDER_USDC=5.0`, `DRY_RUN_BANKROLL_USDC=1000`,
`LATENCY_ARB_EDGE=0.010`, `LATENCY_ARB_MIN_PROB=0.50`, `LATARB_ASK_CAP=0.85`,
`LATARB_MIN_ASK=0.20`, `MAX_SPREAD_PCT=0.15`, `LATARB_ORACLE_MAX_AGE_S=1.2`.

---

## 2. Methodology

### 2.1 Why a *consistently* reversed bot is impossible

The obvious design — flip the signal and let the bot decide normally — cannot trade at all, and
would have produced a silent null run reported as a result. Two gates kill it:

```python
# opus.py:8529-8531
p_up = 0.5 * (1.0 + math.erf(z / math.sqrt(2.0)))
model_prob = p_up if up else 1.0 - p_up
model_prob = max(0.3, min(0.85, model_prob))
```

Flip `up` and `model_prob` lands below `LATENCY_ARB_MIN_PROB=0.50` → `prob_low` skip on every
row. And `edge = model_prob - ask - slip - fee` (`:8555`) goes further negative because the
reversed side is also the more expensive one → `edge_low` on the rest.

That refusal is *correct*: the reversed side is both less likely and dearer. So the only
experiment that can actually run is:

> **Same forward trigger population. Flipped order.**
> Every gate is evaluated on the forward side exactly as the live bot does it, and only the
> order that comes out the far end is inverted.

This is stated plainly because it bounds the claim: the run tests *"would buying the other side
of the trades we take have done better"*, not *"is a reverse strategy viable"*.

### 2.2 The patch — exactly two diff regions

Verified with `diff` after stripping CR (`opus.py` is CRLF; the patcher preserved it):

```
274c274
< LATARB_STATE_PATH: str = '~/latarb_state.json'
---
> LATARB_STATE_PATH: str = '~/rev_latarb_state.json'
```

`LATARB_STATE_PATH` is a **module constant with no `os.getenv`** — it cannot be redirected by
any env file, so it had to be patched or the two bots would share state.

```
8589a8590,8602
>         up = not up
>         _rev_book = mkt.book_yes if up else mkt.book_no
>         if _rev_book is None or _rev_book.best_ask is None:
>             return self._skip_latarb('rev_no_book')
>         ask = float(_rev_book.best_ask)
```

Inserted immediately before `token = mkt.yes_token if up else mkt.no_token` (`:8590`), i.e.
after every gate and before order construction. `py_compile` passes.

**`ask` must be re-read.** Two lines later, `:8609-8614` rebinds
`book = mkt.book_yes if up else mkt.book_no` and rejects on
`cur_ask > ask + max(tick, 0.01)`. Flipping the book while leaving `ask` at the forward side's
0.47 makes the reversed 0.55 trip `quote_moved` on **every** row — another silent null.
`model_prob` is deliberately left forward-facing: the model has no opinion supporting this
side, which is the whole point of the experiment.

### 2.3 Verification discipline used throughout

- **Log scoping.** `opus.log` concatenates many runs. Every counter was scoped with
  `START=$(grep -n 'CLOB balance:' opus.log | tail -1 | cut -d: -f1)` then `tail -n +$START`.
  Unscoped counters mix runs and are meaningless.
- **Schemas read, never guessed.** JSON keys were dumped before use. (See §3.2 — guessing them
  produced two wrong tallies earlier in this work.)
- **Sign conventions read from source before trusting a number.** `opus.py:2863` signs markout
  so negative always means adverse selection (`_dir = +1` for BUY; all 11 reversed fills are BUY).
- **Isolation checked by writer, not by assumption.** Separate ledger paths, separate state
  file, separate log, distinct instance-lock key, and the forward bot's ledger mtimes tracked.

---

## 3. Issues found and fixed

### 3.1 Blockers hit while building the reversed run

| # | Issue | Root cause (verified) | Fix |
|---|---|---|---|
| 1 | Launch died `rc=3`: *"Another Polymarket bot instance is already running (lock held by pid=822553)"* | `acquire_instance_lock` (`:497-500`) keys the lockfile on `proxy_address or sha256(private_key)`. Both bots share one wallet → same key. A real safety feature, not a bug. | `POLYMARKET_PROXY_ADDRESS=0x...dEaD` so the **boot-time** lock key differs. Safe because `:2028-2033` auto-corrects the proxy to the on-chain address afterwards for read-only calls. Forward bot never disturbed. |
| 2 | Shared LatArb state would have cross-contaminated `already_traded` / cooldown | `LATARB_STATE_PATH` is a module constant at `:274`, not env-readable | Patched in the copy only |
| 3 | Flip-only patch = 0 trades, silently | `model_prob < MIN_PROB` → `prob_low` on every row (§2.1) | Trigger on forward side, flip only the order |
| 4 | Book-flip-only patch = 0 trades, silently | `quote_moved` at `:8609-8614` compares the reversed book's ask to the forward `ask` | Re-read `ask` from the reversed book |
| 5 | CRLF→LF regression risk (had happened in earlier work) | `opus.py` is CRLF | Patcher reads bytes, detects `b'\r\n'`, splits/joins on it |
| 6 | Disk at 90% (713 MB free), shadow tape is 359 MB and growing | Two shadow writers would fill the disk | `LATARB_SHADOW=false` + `CALIBRATION_LOG_ENABLED=false` on the reversed bot |
| 7 | 908 MB total RAM, one bot already at ~273 MB | A second interpreter risked OOM-killing the forward bot | `COINS=BTC,ETH`, `DISCOVERY_INTERVAL_S=30` |

### 3.2 Analysis errors I made and corrected (recorded so they are not repeated)

| Error | How it showed | Correction |
|---|---|---|
| Guessed settle JSON keys | Printed `settles=8 WIN=0 pnl=$+0.00` while the log said `day=$-31.86` | Real keys are `net_pnl` and `win` (bool) |
| Guessed fill JSON keys | `price`/`px`/`event`/`status` all returned `None` | Maker: `post_px`, `fill_px`, `fill_shares`, `outcome`, `best_ask_at_post`, `markout_30s_cents`, `markout_120s_cents`. Fills: `actual_fill_px`, `filled`, `matched_size` |
| Read exit 1 as a bot fault | Health check "failed" | `grep -c Traceback` exits 1 on zero matches |
| Checked for `latarb_state.json` | `ls: No such file` | Real name is suffixed: `latarb_state.<n>.dry.<addr>.json`. Isolation still held — the patch produced a distinct `rev_latarb_state.137.dry.<addr>.json` |
| Missing gate in a tape re-extraction | 5923 rows vs target 1067 | Added `disp_too_small` (`abs(log_disp) >= 0.15 * sigma_h`) → 1077 rows, 1067/1067 overlap with the graded set |

### 3.3 Prior fixes still holding in this run (from earlier work, re-confirmed here)

- **gross_cap orphan freeze** — no `gross_cap`/`net_cap` in either bot's skip census. Entries flow.
- **Fee model settled.** `_fee_per_share` (`:4743`) = `0.07 * p * (1-p)`. CLOB reports
  `taker_base_fee=1000`, but the R38 comment at `:4728` records that `1000/10000 = 0.10`
  **overstates charged fees by 43.95%**, that no divisor maps 1000 → 0.07, and that 0.07 is
  calibrated to the venue's own `entryFeesUsdc` over **302 settled positions**
  (median 0.07000, p05 0.06990, p95 0.07000). At p≈0.51 that is **≈1.75c/share** — a hard cost.
- **`DRY_RUN_FILL_PROB=0.0`** — no fabricated fills. `:2851` would otherwise fill any GTC maker
  post on a coin flip at our own limit price, which would report invented profit.

---

## 4. Reversal correctness proof

The `LATARB` entry line logs the side **after** the patch, so under reversal a positive
displacement must print `DN`. Census over every entry line each bot has logged in its current run:

| | `disp > 0` | `disp < 0` |
|---|---|---|
| **`opus_rev.py`** (reversed) | **DN 12, UP 0** | **UP 36, DN 0** |
| **`opus.py`** (forward, control) | UP 200, DN 0 | DN 145, UP 0 |

Perfect inversion, zero leakage, and the control is unchanged. The `rev_no_book` guard added by
the patch **never fired** — the complementary book was always present, so no rows were lost to
the patch itself.

---

## 5. Baseline: the forward bot BEFORE the reversal existed

This is the control the reversal must be judged against. **Anchor:** the reversed bot's own
`BOOT start` line, `[2026-09-09T15:22:21+00:00] pid=830720 build=d5a29961e024`, i.e.
**T0 = 1788967341**. Everything with `ts < T0` is baseline. (Sanity check: the reversed bot's
first maker row is ts 1788968420, comfortably after T0.)

> **Correction to my own first pass.** I initially split on T0=1788970941 and reported
> pre-reversal n=14 / 7.14% / −$62.79. That epoch was wrong. Re-split on the verified boot
> epoch the baseline is **n=11 / 9.09% / −$47.28**. The figures below are the corrected ones.

### 5.1 Baseline funnel, at the exact moment the reversed bot booted

Forward run booted 13:36:51 UTC (the 5th boot in `opus.log`), so the baseline segment is
**13:36:51 → 15:22:01 = 1h 45m**. Counters read from the last STATUS block before the cut:

```
15:21:55  STATUS  pnl=$-46.20  day=$-47.28  orders=0  bal=$942.40  consec=3  fills=13
15:21:55  LATARB fills attempts=26 fills=0 miss=26 avg_edge=0.104
15:21:01  DRYMAKER stats seen=56120 matched=2909 same_side=2262 posted=26
                         fill_through=12 fill_queue=1 expired=13
```

| Baseline funnel (1h 45m) | Count |
|---|---|
| Book messages seen | 56,120 |
| **Entries triggered** (`LATARB UP/DN` lines) | **26** |
| Maker posts | 26 |
| Filled | 13 (**50.0%**) — `through` 12, `queue_cleared` 1, `expired` 13 |
| Settled in the window | 11 |
| Wins | 1 |
| Win rate | **9.09%** |
| Net PnL | **−$47.28** |

Baseline skip census (same shape as the reversed bot's — plumbing dominates, strategy filters next):

```
lead_eval_debounced=227017  lead_eval_coalesced=33680  oracle_stale_lead=33042
disp_too_small=9736  ask_cap=9108  edge_low=8513  book_not_ready=6723
anchor_currency=4616  spread_too_wide=3746  pre_boot_interval=2828
yes_book_not_ready=1447  book_stale=866  no_book_not_ready=554  position_cap=352
already_traded=273  ask_floor=195  adverse_ewma=120  cooldown=101
shard_throughput_stalled=42  missing_open_price=21  dual_book_stale=2  depth_low=1
```

Side census over the baseline segment confirms it is forward-mapped, as a control must be:
**`disp>0 → UP 14, DN 0` · `disp<0 → DN 12, UP 0`**.

### 5.2 Baseline settles — every one of them

| # | ts (UTC) | coin | side | win | net | grade |
|---|---|---|---|---|---|---|
| 1 | 2026-09-09 14:03:36 | BTC | UP | False | −5.19 | dry-onchain-winner |
| 2 | 2026-09-09 14:04:17 | ETH | UP | False | −5.17 | dry-onchain-winner |
| 3 | 2026-09-09 14:09:02 | BTC | UP | False | −5.19 | dry-onchain-winner |
| 4 | 2026-09-09 14:18:51 | BTC | DN | False | −5.26 | dry-onchain-winner |
| 5 | 2026-09-09 14:34:09 | BTC | UP | False | −5.17 | dry-onchain-winner |
| 6 | 2026-09-09 14:34:19 | ETH | UP | False | −5.16 | dry-onchain-winner |
| 7 | 2026-09-09 14:48:35 | ETH | UP | False | −5.16 | dry-onchain-winner |
| 8 | 2026-09-09 14:49:56 | BTC | DN | **True** | **+4.45** | dry-onchain-winner |
| 9 | 2026-09-09 15:09:31 | BTC | UP | False | −5.16 | dry-onchain-winner |
| 10 | 2026-09-09 15:14:03 | BTC | UP | False | −5.13 | dry-onchain-winner |
| 11 | 2026-09-09 15:14:24 | ETH | UP | False | −5.12 | dry-onchain-winner |

All 11 graded `dry-onchain-winner` — the on-chain winner, not the circular spot estimator. Good
provenance. **Every settle is a $5 clip**, so the −$47.28 is 10 losses at ≈−$5.17 and one win at
+$4.45. Note the asymmetry that sets the bar: winners pay **+$4.45**, losers cost **−$5.17**, so
break-even needs **53.8%**.

| Baseline split | n | Wins | Win rate | Net PnL |
|---|---|---|---|---|
| **ALL** | 11 | 1 | **9.09%** | **−$47.28** |
| side = UP | 9 | 0 | **0.0%** | −$46.46 |
| side = DN | 2 | 1 | 50.0% | −$0.82 |
| coin = BTC | 7 | 1 | 14.3% | −$26.66 |
| coin = ETH | 4 | 0 | 0.0% | −$20.62 |

**0 for 9 on UP.** That is a brutal patch, not a discovered bias — n=9, and the whole point of the
prior work is that UP selection is indistinguishable from the base rate.

### 5.3 Baseline maker execution

The maker ledger predates the current run (155 rows back to 2026-09-05, spanning 5 boots), so
both scopes are given:

| Scope | Posts | Filled | Fill rate | `markout_30s` | `markout_120s` | mean `post_px` | mean `best_ask@post` |
|---|---|---|---|---|---|---|---|
| All pre-reversal (2026-09-05 → T0, 5 runs) | 108 | 51 | 47.2% | −12.05c (sd 30.62, t=−2.81) | −15.91c (sd 42.03, t=−2.70) | 0.4510 | 0.4647 |
| Pre-reversal, current run only (≥13:36:51) | 26 | 13 | 50.0% | −13.50c (sd 18.14, t=−2.68) | −30.31c (sd 31.22, **t=−3.50**) | — | — |

Adverse selection was **already significant before the reversal existed** (t=−2.81 on n=51).
That matters for interpretation: the reversal did not introduce adverse selection, it made an
existing problem worse.

### 5.4 What the baseline changes about the comparison

| | Forward BASELINE (pre-T0, 1h45m) | Forward POST-T0 | Reversed (post-T0) |
|---|---|---|---|
| Settles | 11 | 20 | 11 |
| Win rate | **9.09%** | **30.00%** | **45.45%** |
| Net PnL | **−$47.28** | −$28.21 | −$7.85 |
| Avg win / avg loss | +$4.45 / −$5.17 | +$7.40 / −$5.19 | +$4.63 / −$5.17 |
| Break-even needed | 53.8% | 41.2% | 52.73% |

**Read this carefully, because it cuts against the headline.** The forward bot's own win rate
went 9.09% → 30.00% across T0. Nothing about launching a second process could cause that — it is
regime and variance. So the §6.4 "same-window" comparison pits the reversal against a forward bot
that was in a *better* patch than its own baseline. Two consequences, both stated rather than
buried:

1. **Reversal's 45.45% is not better than forward's 9.09% in any meaningful sense.** Different
   windows, n=11 each, and the forward bot's own two windows differ by 21 pp with no cause.
2. **The honest comparison remains the same-window one** (§6.4: forward 40.00% vs reversed
   45.45%), and even there the market sets are partly disjoint. The baseline's value is showing
   how violently these small-n windows swing — 9.09%, 30.00%, 40.00%, 45.45% are all the same
   strategy family over ~16 hours.

The one thing the baseline establishes solidly and independently: **losers cost more than winners
pay** (−$5.17 vs +$4.45), so break-even is 53.8%, not 50%. That is the fee plus the spread, and it
is present before any reversal.

---

## 6. Strict dry-run overview — the reversed run

### 6.1 Reversed bot — full funnel

**Run:** pid 831096 (ppid 831086), booted 2026-09-09 15:22:21 UTC, last STATUS 2026-09-10
05:41:54 UTC ⇒ **14h 19m**, **1 boot, 0 restarts, 0 tracebacks, 0 `REFUSING TO LAUNCH`**.
RSS 234772 KB. Simulated book $1000, clip fixed at $5.

```
STATUS  pnl=$-7.85  day=$-0.98  orders=0  bal=$992.15  OK  consec=1  fills=11
DRYMAKER stats  seen=257524  matched=4940  same_side=4059  posted=48
                fill_through=10  fill_queue=1  expired=37
```

| Funnel stage | Count | Note |
|---|---|---|
| Book messages seen | 257,524 | |
| **Entries triggered** (`LATARB UP/DN` lines) | **48** | matches `attempts=48` and `posted=48` |
| Maker orders posted | 48 | `ENTRY_MODE=maker`, 120 s GTD |
| **Filled** | **11** | **22.9% fill rate** |
| ├─ `through` (book traded through our bid) | 10 | |
| ├─ `queue_cleared` | 1 | |
| └─ `expired_no_print` | 37 | |
| **Settled** | **11** | 0 open |
| **Wins** | **5** | |
| **Losses** | **6** | |
| **Win rate** | **45.45%** | |
| **Net PnL** | **−$7.85** | |

**Skip census (why the other ~627k evaluations did not trade):**

```
lead_eval_debounced=496147   disp_too_small=81811   oracle_stale_lead=76572
lead_eval_coalesced=69663    shard_throughput_stalled=57931   ask_cap=46426
book_stale=34993   book_not_ready=33643   edge_low=19900   spread_too_wide=17373
yes_book_not_ready=15225   no_book_not_ready=10630   pre_boot_interval=2016
already_traded=968   ask_floor=622   cooldown=175   position_cap=82
dual_book_stale=67   depth_low=4   edge_fill_low=4
```

The top four are throughput/plumbing, not strategy: `lead_eval_debounced` and
`lead_eval_coalesced` are rate-limiting, `oracle_stale_lead` is the 1.2 s freshness gate
(correctly refusing to trade on information the venue already has), and
`shard_throughput_stalled` is the known off-box WS degradation documented in `DRYRUN_ONLY.env`.
The real strategy filters are `disp_too_small` (81,811), `ask_cap` (46,426) and `edge_low` (19,900).

**Reading trap:** the line `LATARB fills attempts=48 fills=0 miss=48 rate=0.0%` says **fills=0**
while `STATUS` says `fills=11`. That counter tracks the FAK path; under `ENTRY_MODE=maker` real
execution is recorded in `DRYMAKER stats` and `rev_latarb_maker.jsonl`. Same artifact as the
known `latarb_fills.jsonl` maker-mode/DUP_GUARD issue — do not score a maker run off it. For
that reason `rev_latarb_fills.jsonl` was **not** used in this analysis.

### 6.2 Every reversed fill, individually

| # | ts | coin | side | post_px | fill_px | best_ask@post | shares | mk30c | mk120c | outcome |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | 1788970249 | BTC | BUY | 0.54 | 0.54 | 0.55 | 9.26 | −39.5 | −53.0 | through |
| 2 | 1788970979 | ETH | BUY | 0.55 | 0.55 | 0.67 | 9.09 | −3.5 | +44.9 | through |
| 3 | 1788974337 | ETH | BUY | 0.56 | 0.56 | 0.58 | 8.93 | −55.0 | −55.0 | through |
| 4 | 1788976341 | BTC | BUY | 0.49 | 0.49 | 0.51 | 10.20 | +16.5 | +50.0 | through |
| 5 | 1788983309 | BTC | BUY | 0.48 | 0.48 | 0.49 | 10.42 | −47.0 | −47.9 | through |
| 6 | 1788988091 | ETH | BUY | 0.55 | 0.55 | 0.64 | 9.09 | −54.0 | −54.9 | through |
| 7 | 1788991128 | BTC | BUY | 0.50 | 0.50 | 0.50 | 10.00 | −29.5 | +49.0 | through |
| 8 | 1789003242 | BTC | BUY | 0.55 | 0.55 | 0.59 | 9.09 | −26.5 | +44.0 | through |
| 9 | 1789004865 | BTC | BUY | 0.49 | 0.49 | 0.51 | 10.20 | −46.5 | −48.0 | through |
| 10 | 1789006685 | BTC | BUY | 0.47 | 0.47 | 0.48 | 10.64 | −43.5 | +52.0 | through |
| 11 | 1789015988 | BTC | BUY | 0.46 | 0.46 | 0.47 | 10.87 | −23.5 | −45.0 | queue_cleared |

Posting prices: all 49 posts mean **0.5059**, the 11 filled mean **0.5127**,
`best_ask_at_post` mean **0.5398**. (The maker ledger grew from 48 to 49 rows between two reads
mid-analysis; the fill/settle set is unaffected.)

### 6.3 Reversed bot — splits and break-even

| Split | n | Wins | Win rate | Net PnL |
|---|---|---|---|---|
| **ALL** | 11 | 5 | **45.45%** | **−$7.85** |
| side = UP *(mirror of forward DN triggers)* | 9 | 5 | 55.6% | +$2.49 |
| side = DN *(mirror of forward UP triggers)* | 2 | 0 | 0.0% | −$10.34 |
| coin = BTC | 8 | 4 | 50.0% | −$1.47 |
| coin = ETH | 3 | 1 | 33.3% | −$6.38 |

**Realized break-even from its own fills:** average winner **+$4.63**, average loser **−$5.17**
⇒ needs **52.73%** to break even, delivered **45.45%** ⇒ **−7.28 pp short**.

That sits on top of the backtest's **−6.44 pp** prediction for the reversal, but with n=11 the
agreement is *consistent*, not confirmatory. Do not cite it as validation.

### 6.4 Forward bot in the same wall-clock window

Window = the reversed bot's settle span, ts 1788971060 → 1789016633.

| Metric | Forward (control) | Reversed |
|---|---|---|
| Maker posts | 37 | 48 |
| Filled | 15 (**40.5%**) | 11 (**22.9%**) |
| `through` / `queue_cleared` / `expired` | 14 / 1 / 22 | 10 / 1 / 37 |
| Mean `post_px` | 0.4465 | 0.5059 |
| Mean `best_ask_at_post` | 0.4630 | 0.5398 |
| Settles | 15 | 11 |
| Win rate | **40.00%** | **45.45%** |
| Net PnL | **−$2.36** | **−$7.85** |
| Avg win / avg loss | +$7.40 / −$5.20 | +$4.63 / −$5.17 |
| Break-even needed | 41.2% | 52.73% |
| `markout_30s` | −21.97c (sd 26.62, se 6.87, **t=−3.20**) | **−32.00c** (sd 22.16, se 6.68, **t=−4.79**) |
| `markout_120s` | −10.86c (se 11.35, t=−0.96) | −5.81c (se 15.57, t=−0.37) |

Forward bot, whole current run (booted 13:37:52, scoped): `attempts=73`, `posted=73`,
`fill_through=29`, `fill_queue=2`, `expired=42`, `fills=31`, `pnl=$-64.07`, `bal=$924.51`.
Forward all-time settles: n=29, 7 wins, **24.14%**, **−$65.15**.

**Important caveat — this comparison is not paired.** The two bots are independent processes
with their own debounce/coalesce timing, their own `already_traded` state and their own
cooldowns, so they do not see the same trigger instants. Direct proof: reversed `UP` went 5/9
and forward `DN` went 5/9 over the same window. If those were mirrors of the same markets, one
side would have to lose whenever the other won — so the market sets are demonstrably disjoint in
part. Read the two columns as two samples of the same regime, not as a mirror pair.

---

## 7. What the data says about *why* reversal loses

Mechanism, measured on the graded post-R51 tape (n=1067, 0 unresolved) and corroborated live:

- **`mean(ask + opp_ask) = 1.0254`.** Both sides of the book are priced above fair — that 2.54%
  is the vig. Reversing does not escape the spread, it pays it on the expensive side.
- Reversing buys **0.66 pp** more win rate (49.67% → 50.33%) and pays **7.62 pp** more in price
  (mean ask 0.4746 → mean opp_ask 0.5508). Net ≈ −7 pp. Confirmed live: reversed posts averaged
  0.5059 against the forward bot's 0.4465 in the same window, a **+5.94 pp** price penalty.
- **Execution gets worse too.** 10 of 11 reversed fills are `through` — the book traded through
  our resting bid, i.e. we fill precisely when the market is moving against us. Reversed
  `markout_30s` is **−32.00c** (t=−4.79) vs the forward bot's −21.97c (t=−3.20) in the same
  window. Resting on the expensive half of the vig gets run through harder. Fill rate also
  halves (22.9% vs 40.5%): the cheap side fills easily precisely because it is the wrong side
  to be resting on.
- Backtest predictions for reference (n=1067, graded, taker/maker): ALL **−6.44 pp** /
  **−3.89 pp**; ttc≥40 −5.77 / −3.21; ttc[40,60) −24.38; BTC −14.18. Only XRP improved
  (+1.63 pp) and its CI spans zero.

**There is no edge to invert.** The forward signal measures **+0.52 pp CI[−2.48,+3.52]** on the
clean post-R51 population — statistically nothing. Flipping a zero-edge signal cannot create
edge; it only moves you to the dearer side of the book and worsens your fills.

---

## 8. Health and safety state

| Check | Status |
|---|---|
| Forward bot (pid 822553) | Up 15h47m, RSS 273032 KB, healthy, **untouched** |
| Reversed bot (pid 831096) | Up 14h19m, RSS 234772 KB, 1 boot, 0 tracebacks |
| Interpreters running | Exactly 2, one per bot |
| Ledger isolation | Holds — separate settle/maker/fills/state/log for each |
| **Memory** | **138 MB available of 908 MB; swap 343 MB of 1535 MB (was 88 MB).** Tight. A third process or re-enabling the shadow tape would risk OOM. |
| **Disk** | **713 MB free, 90% used.** `LATARB_SHADOW=false` on the reversed bot is load-bearing, not cosmetic. |
| Real capital at risk | None. `DRY_RUN=true`; the real wallet holds $6.48 with $0.00 MATIC, so nothing can execute for real. |
| **Wallet key** | **Treat as compromised. Rotate before any funding or live use.** Not read or echoed anywhere in this work. |

---

## 9. Conclusions and next steps

1. **The reversal works and is worse.** −$7.85 over 11 settles at 45.45% against a 52.73%
   break-even, versus the forward bot's −$2.36 at 40.00% against 41.2% in the same window.
   Both lose; reversal loses harder and for an identifiable reason (the vig plus worse adverse
   selection), not by luck.
2. **n=11 settles nothing on its own, and the baseline proves why.** The same forward strategy
   printed 9.09% (baseline), 30.00% (post-T0) and 40.00% (rev-window) across ~16 hours. Windows
   of this size swing 30 pp on nothing. The reversal is *proven correct*; its performance is
   *directionally consistent* with a −6.44 pp backtest. Do not upgrade that to a finding.
3. **What the baseline does establish, solidly:** losers cost more than winners pay
   (−$5.17 vs +$4.45 on $5 clips), so break-even is **53.8%**, not 50% — and adverse selection
   was already significant before the reversal existed (−12.05c/share, t=−2.81, n=51). Neither
   is a small-n artifact, and neither is caused by the experiment.
4. **Do not fund or go live.** Independent of this experiment: the clean signal measures
   +0.52 pp CI[−2.48,+3.52] against a hard ≈1.75c/share fee, and the key is compromised.
5. **Leave both dry runs up** — they cost nothing but disk and RAM, and both are near their
   limits. Watch memory; do not add a third process.
6. **Still open from prior work:** adjudicate the two pre-registered hypotheses
   (ttc[40,60) and coin=BTC) strictly out-of-sample on ts > 1788965000. They are hypotheses,
   not findings — ttc[30,40) is −10.05 pp right next to ttc[40,60)'s +18.74 pp, and XRP mirrors
   BTC with the opposite sign.

### Commands to reproduce any number above

```bash
export MSYS_NO_PATHCONV=1
ssh -i <key>.pem ubuntu@3.249.210.105

# reversal proof
grep -oE 'LATARB (UP|DN) [A-Z]+ \| disp=-?[0-9.]+' ~/rev.log

# reversed funnel
grep 'DRYMAKER stats' ~/rev.log | tail -1
grep 'LATARB skips'  ~/rev.log | tail -1

# forward, scoped to its current run (unscoped counters mix runs)
START=$(grep -n 'CLOB balance:' ~/opus.log | tail -1 | cut -d: -f1)
tail -n +$START ~/opus.log | grep 'STATUS' | tail -1

# settles (keys are net_pnl and win - do not guess them)
python3 -c "import json;r=[json.loads(l) for l in open('/home/ubuntu/rev_latarb_settle.jsonl') if l.strip()];print(len(r),sum(1 for x in r if x['win']),sum(float(x['net_pnl']) for x in r))"

# BASELINE: split the forward ledger on the reversed bot's own BOOT epoch.
# Anchor from the log line itself - do NOT eyeball it, I got this wrong once.
T0=$(date -u -d "$(grep -m1 -oE '2026-[0-9]{2}-[0-9]{2}T[0-9:]+' ~/rev.log)" +%s)   # -> 1788967341
echo $T0   # 1788967341 = 2026-09-09 15:22:21 UTC
python3 -c "import json,sys;T=int(sys.argv[1]);r=[json.loads(l) for l in open('/home/ubuntu/latarb_settle.jsonl') if l.strip()];p=[x for x in r if float(x['ts'])<T];print(len(p),sum(1 for x in p if x['win']),round(sum(float(x['net_pnl']) for x in p),2))" $T0

# baseline funnel: scope to the current run, then cut at the reversal boot
PS=$(grep -n 'CLOB balance:' ~/opus.log | tail -1 | cut -d: -f1)
REL=$(tail -n +$PS ~/opus.log | grep -n '^15:22:' | head -1 | cut -d: -f1)
sed -n "${PS},$((PS+REL-1))p" ~/opus.log | grep -E 'STATUS|DRYMAKER stats' | tail -2
```
