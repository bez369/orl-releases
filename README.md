# ORL

ORL is an automated trader for Pons V2 launches on Robinhood Chain.

The project is now in Phase 2 paper-trading mode for Pons V2 on Robinhood Chain (chain id `4663`). It watches the factory's `TokenLaunched` event, records what it enters in SQLite, and simulates curve entries and exits against live on-chain reads. It does not load a private key, submit transactions, or place trades.

## Fees

ORL is free to use. To support development, it charges a **usage fee of 1% of the ETH spent on each buy** while live trading. The fee is sent as its own on-chain transfer, right after each buy confirms, to the developer address `0x48D0eEE30ec5BF5C7E167036a8c7f3e4820c19fa`. Paper trading and live dry runs send nothing. Sells and approvals carry no usage fee (you still pay the network's gas on every transaction). The fee is shown again, with this address, on the "go live" screen before any live trading starts.

The distributed app is bundled and hardened to protect the code, but it is unsigned and runs locally, so a determined user can still modify it — this is a deterrent, not DRM.

## Desktop app (Windows, Linux)

The same engine also ships as a desktop app with a GUI: a dashboard (open positions with Sell/Skip, closed trades, session P&L, curve/launch/quote panels, RPC health, logs), a settings form, a wallet manager, and start/stop with paper/live switching. The headless server setup below is unchanged.

- **Install**: download `ORL-Setup-<version>.exe` (Windows) or the `.AppImage` / `.deb` (Linux) from the GitHub release. Windows builds are not code-signed yet, so SmartScreen warns on first run ("More info → Run anyway").
- **First run**: accept the risk notice, then paste your own RPC URL (it is tested for chain 4663), then optionally create or import a wallet. The app always starts in **paper** mode.
- **Settings**: the form is generated from `.env.example` (basic) and `.env.advanced.example` (advanced), and the strategy picker sets the entry lane (A curve-close, B/C bundle-router, D bundle-service). Every save is checked by loading `src/config.ts` with the new values, so the app refuses exactly what the server would.
- **Wallet**: the key is stored encrypted with your password (Web3 Secret Storage v3, the same keystore format other wallets import). It is decrypted in memory only when you go live, which needs the password and typing `LIVE` each time. A live dry run is the default.
- **Where things live**: `~/.config/ORL` on Linux, `%APPDATA%\ORL` on Windows (an upgrade from the old "Pons Sniper" build copies its folder over on first start). That folder holds `settings.json`, `wallet.json`, `data/` (the SQLite database), and `logs/`.
- **Quitting** stops the bot. Open live positions stay in the wallet and are no longer watched.

Build from source (Node 22+):

```sh
npm install
npm run desktop:dev      # build everything and open the app
npm run desktop:dist     # installers for this OS into release/
```

Pushing a `v*` tag runs `.github/workflows/desktop-release.yml`, which builds the Windows and Linux installers on their own OSes and attaches them to a draft GitHub release. Once the release is published, installed apps update from it. The repository, or a separate public releases repository set in `electron-builder.yml`, must be public for downloads and updates to work without a token.

## Setup

Requires Node.js 22+ and an RPC provider with an HTTPS endpoint. HTTP polling is used for reliable event detection; `RPC_WS_URL` is reserved for a later low-latency transport.

The RPC URL must contain your real provider key. For Alchemy, use `.../v2/YOUR_KEY`, without braces or other placeholder characters.

```sh
cp .env.example .env
npm install
npm run build
npm start
```

`.env.example` holds the settings you tune: connection, mode, entry path, sizes,
the curve-close trigger and its filters, exits, alerts. Everything in it has a
default, so any line can be deleted. The tuning knobs — RPC hardening, feed and
tracker internals, polling cadence, live-execution bounds, and the launch /
graduation / whitelist lanes — are in `.env.advanced.example` with their
defaults; append a line from it only when you mean to change it. `RPC_HTTP_URL`
is the one setting with no default.

Set `START_BLOCK` to a decimal block number for a historical backfill, or leave it as `latest` to observe new events. The database is created at `DATABASE_PATH`.

`MAX_EVENT_AGE_MS` (default 30s, `0` disables) is the staleness guard: a launch or graduation whose **block** is older than this is logged but never entered. A backfill, a cold `START_BLOCK`, or a long RPC backoff all replay historical events down the same path as live ones, and an entry reads live reserves — so without the guard the bot buys an hours-old token at today's price. Age comes from the block timestamp, not from when the process noticed.

All three entry paths are off by default. Set `PRE_GRADUATION_ENTRY=true` to take bonding-curve positions, `GRADUATION_ENTRY=true` to take post-graduation v4-pool positions, and/or `CURVE_CLOSE_ENTRY=true` to take closing shots on nearly full curves. They are independent: enable any combination.

Each flag gates its **entire pipeline**, not just the entry. With a flag off, the corresponding listener never starts, so its polling, metadata reads and database writes stop as well. Three consequences:

- With all three flags false nothing would start, so the bot refuses to boot. There is no observe-only mode.
- `LIVE_TRADING=true` trades **one** path per process: `CURVE_CLOSE_ENTRY=true` **or** `ROUTER_BUNDLE_ENTRY=true`. Both on together is refused — a trade claim is unique per `(token, strategy)`, so two live lanes on one wallet could buy the same token twice, and a bag one lane kept (`MAX_HOLD_ACTION=skip`, `/skip`) would be sold by the other's exit against the wrong cost basis. `PRE_GRADUATION_ENTRY` and `GRADUATION_ENTRY` are refused live at all, because launch entries would otherwise buy with the live wallet. Run the other paths' paper trading in a separate process.
- `PRE_GRADUATION_ENTRY=false` stops the `launches` table filling and the launch listener's in-memory cache from ever populating; graduations then read their metadata from the chain. Graduation-only sniping is `PRE_GRADUATION_ENTRY=false` + `GRADUATION_ENTRY=true`.

Setting `PAPER_TRADE=false` still suppresses simulated positions while the listeners run.

### Entry timing: only the snipe tax is waited for

A launch entry waits for one thing: the snipe tax. Robinhood Chain has no gas-priority queue, because the sequencer orders strictly by arrival. So every other millisecond spent "confirming" a launch is fill quality handed to someone who didn't wait.

The tax is charged on the recipient and decays in whole-second steps of block time. Measured on 2026-09-14 it was 9900 bps in the launch second, 618 at +1 s, 19 at +2 s, and 0 from +3 s. A launch is detected well inside that window.

The entry therefore polls the curve's live `currentSnipeTaxBps` every `LAUNCH_SNIPE_TAX_POLL_MS` (default 150). As soon as the tax is at or under `MAX_SNIPE_TAX_BPS`, it reads and buys in one pass. It never assumes a decay curve. With the default `500` the buy lands at about +2 s; `700` would buy at about +1 s and pay 6.18%. A launch whose tax hasn't fallen within `LAUNCH_SNIPE_TAX_MAX_WAIT_MS` (default 10000) is given up (`snipe_tax_timeout`). While it waits, the launch holds its position slot.

Before this, the launch path read the tax once and skipped any launch above the cap, which was nearly every launch. Also removed: the fixed `ENTRY_DELAY_MS` sleep, the feed-nudge minimum interval, and the RPC in-flight concurrency cap (`RPC_MAX_CONCURRENCY`). Each one queued the entry behind elapsed time or other traffic.

Curve-entry defaults are: snipe tax at or below 5%, one open position, and a force-close after five minutes. Pairs: native ETH, plus on paper any stock- or USDG-quoted launch with a Chainlink feed (`LAUNCH_STOCK_QUOTES`, default true). The `PAPER_QUOTE_IN_WEI` size is swapped into the quote at the oracle price less `CURVE_CLOSE_QUOTE_SWAP_COST_BPS`, and back on exit, so P&L stays in ETH, the same model as stock curve-close shots. A pair with no feed skips as `unsupported_pair`, and one whose feed is stale skips as `no_quote_price`. Live launch entries stay native-ETH only. With no launch filter on, every launch that meets those conditions is entered. At the first take-profit it sells enough tokens to recover the initial quote investment, or, with `TAKE_PROFIT_SELL_PCT` set, that percent of the tokens bought (`90` sells 90% of the bag). The listener keeps the remainder open until the second take-profit, stop-loss, graduation, or maximum hold. Every sale is valued net of the pons fee and creator tax. A trade's P&L is the sum of all its exits: the `SELL` line shows the exit's own `pnl` and, for a staged trade, the `trade so far` / `trade total`. The dashboard's position row shows the whole trade, and the status bar counts wins and losses per trade. Adjust `MAX_HOLD_MINUTES` in `.env`.

### Launch filters (optional, launch entry only)

These filters gate `PRE_GRADUATION_ENTRY`. Each has its own switch. With several on, a launch is bought only if it passes **all** of them, and anything they can't establish skips it.

For launch-only sniping with all three filters, set:
- `PRE_GRADUATION_ENTRY=true`
- `GRADUATION_ENTRY=false`
- `CURVE_CLOSE_ENTRY=false`
- `LAUNCH_FILTER_OG=true`
- `LAUNCH_FILTER_FRESH_CREATOR=true`
- `LAUNCH_FILTER_DEV_HOLDING=true`

- **OG name and ticker** (`LAUNCH_FILTER_OG`). No earlier Pons launch used the same name, and none the same ticker. Each field is checked on its own.
  - Names are compared by key: NFKC-folded, lowercased, letters and digits only. So `$PEPE`, `pepe`, `P E P E` and `ＰＥＰＥ` are one name, while `Pepe 2` is a different one. A name with no letters or digits (emoji only) is compared on its text.
  - "Earlier" means an earlier block, or the same block with a lower log index.
  - `og=copy(name x3 ticker x12, first PEPE)` names the first launch that used it. One earlier match is enough to skip, even while the index is still backfilling.
  - "Never seen" needs three things: the index complete back to `FACTORY_DEPLOY_BLOCK`, caught up to the launch's own block, and every earlier launch named. The last few seconds of launches may not be named yet, so they are read on demand.
  - Otherwise the result is `og=unresolved(<reason>)`: `index_backfilling_<pct>pct`, `names_pending_<n>`, `name_keys_building` (the first start after upgrading), `no_metadata`, `index_behind`.
  - This filter needs `LAUNCH_INDEX_NAMES=true`.
- **Fresh creator** (`LAUNCH_FILTER_FRESH_CREATOR`). The deployer has never launched before (`creator_not_fresh` otherwise). This is the same lookup as curve-close filter A in `fresh` mode, including `shared_contract_deployer` counting as unresolved.
- **Dev holding** (`LAUNCH_FILTER_DEV_HOLDING`). The dev wallets must hold less than `LAUNCH_MAX_DEV_HOLDING_PCT` (default 20) % of the supply. Dev wallets are the same as curve-close filter C: deployer, launch-tx sender, and launch-tx buy recipients. At launch, "held" is essentially what the launch tx bought.
  - The launch tx comes straight from the launch event, so this filter doesn't need the index.
  - A balance read from a node that hasn't reached the launch block yet counts as unknown (`node_behind`), not as 0%.

- **Minimum dev buy** (`LAUNCH_MIN_DEV_BUY_WEI`, ETH wei, 0 = off). The launch tx's own buys on the curve must have spent at least this much. That is what the dev wallets paid, fees included; a dev buying in a later tx isn't counted.
  - A stock- or USDG-quoted launch's buy is converted to ETH at the Chainlink price, so one setting covers every quote. No fresh price means `dev_buy_unknown`, and the launch is skipped.
  - It reads the same launch-tx receipt as the dev-holding filter; it is fetched once. The result shows as `devBuy=0.06 ETH` or `devBuy=1.2 NVDA≈0.05 ETH`.
  - The OG research of 2026-09-15 found it gamed. Serial spam launchers buy exactly 0.05, 0.12 or 0.2 ETH on a fresh deployer and sell within seconds, and most real runners had a small or no dev buy. Leave it at 0 and use the launch confirm below.

### Launch confirm and bundle split (optional, after the filters)

- **Confirm** (`LAUNCH_CONFIRM_WAIT_SECONDS`, 0 = off, max 120). A launch that passed the filters is bought only at launch block time + N s, and only if both hold:
  - The curve price is at least `LAUNCH_CONFIRM_MIN_RISE_PCT` % (default 0, "hasn't fallen") over its price when the snipe tax first passed.
  - The dev wallets still hold at least `LAUNCH_CONFIRM_MIN_DEV_KEPT_PCT` % (default 90) of what the launch tx bought them.

  Skips show as `no_momentum`, `dev_sold` or `dev_unresolved`. The wait holds no position slot. It is the whitelist's momentum confirm plus a fresh dev-holding read, so it needs a launch filter on, and `LAUNCH_FILTER_DEV_HOLDING=true` unless the kept % is 0. In the research, 49 of 206 passed ETH launches had the dev sell within ~7 s, and a +2 s entry took that dump.
- **Bundle split** (`LAUNCH_BUNDLE_MIN_WALLETS`, 0 = off; needs `LAUNCH_FILTER_DEV_HOLDING=true`).
  - A native-ETH launch whose launch tx bought into at least N dev wallets is paper-traded as `strategy=launch_bundle`.
  - Every other passed launch, stock/USDG bundles included, trades as `strategy=launch_confirm`.
  - `npm run research` compares the two (`bot strategy` split).
  - The one rule positive in both halves of the research was ETH, ≥ 4 wallets and dev holding < 20%: 3 of 3 ran 2.8–3.4× within 90–250 s. That is n = 3, a hypothesis to paper-test, not a proven edge.
- **Launch exits.** `LAUNCH_TAKE_PROFIT_MULTIPLE`, `LAUNCH_SECOND_TAKE_PROFIT_MULTIPLE`, `LAUNCH_TAKE_PROFIT_SELL_PCT`, `LAUNCH_STOP_LOSS_MULTIPLE` and `LAUNCH_MAX_HOLD_MINUTES` apply to launch entries only (untagged, bundle, confirm). Each unset one falls back to the global rule. Curve-close and graduation positions keep the global rules. The research pick is TP 2× selling 100%, with a 5-minute hold.
### Exit policy: `EXIT_STRATEGY=fixed | trail | pyramid`

`fixed` (default) is the ladder above: `TAKE_PROFIT_MULTIPLE`, `SECOND_TAKE_PROFIT_MULTIPLE`, `STOP_LOSS_MULTIPLE`, plus max hold. `trail` (`src/exit-strategy.ts`) replaces every one of those with a path-dependent policy, because the ladder sells a token that is about to 4x at 1.5x, and its short max hold sells a winner that has not moved yet:

- **Unarmed** (under `EXIT_ARM_MULTIPLE`, default 1.3): only `EXIT_RUG_STOP_MULTIPLE` (0.6) and `EXIT_NO_PROGRESS_MINUTES` (15) apply. Most losers never leave this state.
- **Armed:** the trailing stop below is the only stop; the rug stop and no-progress clock are off. There is no break-even floor (removed 2026-09-23): a bundle's first-minute dump swung armed positions through it, and the "break-even" sale filled at a loss after impact while the token went on to run.
- **Cost back:** at `EXIT_COST_BACK_MULTIPLE` (1.5) it sells just enough to recover the entry cost; the rest is free.
- **Moonbag:** the rest rides `EXIT_TRAIL_TIERS` (`0:25,3:15,6:8` — 25% below the peak, 15% once the peak has been 3x, 8% from 6x).
- `EXIT_STALL_MINUTES` (0 = off) exits a position that stops making highs before cost back. Its clocks count only *watched* time, so downtime longer than the window doesn't sell on the first poll after a restart.
- Max hold and `MAX_HOLD_ACTION` still apply. Max hold is checked **before** the policy, so a short one means the trail never runs: `trail` refuses to boot live with `MAX_HOLD_MINUTES` under 30 and warns on paper. It also refuses `EXIT_COST_BACK_MULTIPLE` below `EXIT_ARM_MULTIPLE`, and (live) the rug stop and no-progress clock both off.
- Live, the price-triggered exits (`rug_stop`, `trail_stop`, `giveback_stop`) sell the whole bag on the escalating stop slippage, because chunking a falling bag across 3-second polls costs more than the impact does. The clock exits (`no_progress`, `stall`) and `cost_back` stay impact-capped like a take-profit; the cap only defers the rest to the next poll, which re-triggers while the condition holds.
- The peak is journalled (`live_trades.peak_price/peak_at`, `paper_trades.peak_price/peak_at`), so a restart resumes the same trail instead of re-arming from the price it comes back to. A failed write only costs the trail its memory — it never blocks a sale.

**`npm run backtest`** (`src/research/exit-backtest.ts`) is what those numbers come from. It rebuilds real price paths from logs alone — the curve's `CurveBuy`/`CurveSell` for the pre-graduation leg and progress replay (so the entry is the bot's own `CURVE_CLOSE_TRIGGER_PCT_LEFT` trigger), then PoolManager `Swap` for the pool leg, whose `sqrtPriceX96` and liquidity give both price and reserves with no archive state. Ticks are decimated to `POSITION_POLL_INTERVAL_MS`, sales pay fees, gas and their own price impact, and the window between the closing buy and the pool's first swap is untradable, as it is on chain.

```
npm run backtest -- --hours 36 --horizon-min 120     # fetch (cached under data/exit-backtest) and score
npm run backtest -- --offline --sweep --split        # sweep parameters on the cache, older half vs newer
npm run backtest -- --tokens 0xabc,0xdef --detail    # specific tokens, per-trade lines
```

`--flow` scores the same trail with a rolling 60 s flow window over it (net quote flow and the largest single sell, both as a share of the pool's quote reserve). On that sample it did **not** add P&L: tightening the trail on net outflow was flat (+0.0456 at best), and an immediate exit on one sell worth 3% of the reserve pushed the win rate to 86% while cutting the average to +0.0335 — it sells winners into ordinary size. Only undifferentiated flow was tested; dev/bundle-tagged flow needs the token's `Transfer` logs (a v4 `Swap`'s `sender` is the router, not the trader) and is still untested.

On the 36 h to 2026-09-17 16:00 UTC (90 native-ETH curve-close entries, 0.1 ETH a shot): the ladder then in `.env` (TP 1.5/2, SL 0.7, 3-minute hold) made **+0.0342 ETH a trade**, the shipped `trail` defaults **+0.0449** (+31%), winning on each half separately. Median peak was 2.27x at +6 minutes; 54% of entries reached 2x, 19% reached 5x. 51 of 90 dipped to 0.7x before any 2x and only 10 of those recovered, which is why the rug stop stays. Worked examples from that day: MOUSE -0.040 -> +0.121, 001 +0.006 -> +0.029, MUSEX -0.035 -> -0.055.

#### `pyramid`: cost back, then a back-loaded ladder

`trail` cuts the losers well but closes the winners small: a token on its way to 100k mc pulls back 25-30%, the tiers sell the whole moonbag there, and a 5x ends as a small plus. `pyramid` keeps everything `trail` does up to and including the cost-back leg and replaces the tiers with fixed price rungs — more sold the higher it goes — under a single give-back stop that never tightens as the peak grows.

- **Unarmed and cost back:** identical to `trail`, and driven by the same settings (`EXIT_ARM_MULTIPLE`, `EXIT_RUG_STOP_MULTIPLE`, `EXIT_NO_PROGRESS_MINUTES`, `EXIT_COST_BACK_MULTIPLE`). At 1.5x the position sells just enough to recover the entry cost, about 68% of the bag; what is left is the **moonbag**.
- **Rungs** (`EXIT_PYRAMID_RUNGS`, default `2:10,3:20,5:30,10:40`): sell 10% of the moonbag at 2x, 20% at 3x, 30% at 5x and the last 40% at 10x — more the higher it goes. The position is always sold *down to* the target the current price implies, so a gap that jumps two rungs sells both in one go, and an impact-capped or part-filled sale finishes itself on the next poll. The percentages may add up to less than 100 to keep a permanent runner.
- **Give-back stop** (`EXIT_PYRAMID_GIVEBACK_PCT`, default 25; `EXIT_PYRAMID_GIVEBACK_ARM_MULTIPLE`, default the cost-back multiple, `0` = the first rung): once the peak reaches the arm multiple, a fall of that percentage from the peak sells whatever is left. From there it is the moonbag's only floor. Before it arms, an armed position is guarded by `EXIT_RUG_STOP_MULTIPLE` alone (no break-even floor, and the no-progress clock is off once armed), so live refuses to boot with `EXIT_RUG_STOP_MULTIPLE=0` under any strategy (open `router_bundle_c` positions always resume on the pyramid).
- **"The cost is back" means the sales so far returned the ETH that went in**, not "a partial exit happened": the cost-back leg is impact-capped like any take-profit, and a leg that settles short simply runs again on the next poll instead of opening the rungs.
- The moonbag base is journalled (`live_trades.moonbag_tokens`, `paper_trades.moonbag_tokens`) alongside the peak, so a restart sizes the rungs from the same base. A position that arrives mid-flight (policy switched, or a row from before the column) seeds it from what it holds on its first poll.
- Live, `giveback_stop` is a stop reason (whole bag, escalating slippage, no impact cap); the `pyramid_*` rungs are take-profits and stay impact-capped like `cost_back`.
- Same guards as `trail`: `MAX_HOLD_MINUTES` of at least 30 (hours in practice), a downside exit live (for `pyramid`, a rug stop), arm <= cost back, and the first rung at or above the cost-back multiple.

Scored with `npm run backtest -- --offline --split` on 193 curve-close entries over the four days to 2026-09-20 16:30 UTC (0.1 ETH a shot, 120-minute horizon, sales paying their own impact, fees and gas):

| policy | avg/trade | median | win% | best trade | older half | newer half |
| --- | --- | --- | --- | --- | --- | --- |
| fixed (.env ladder) | +0.02629 | +0.01488 | 58% | +0.87746 | +0.03565 | +0.01703 |
| trail (defaults) | +0.02749 | +0.02230 | 55% | +0.60658 | +0.04395 | +0.01120 |
| **pyramid (defaults)** | **+0.02962** | **+0.02650** | 53% | **+0.87746** | +0.04264 | +0.01672 |

`pyramid` wins on the whole sample and on the median, and it is the only policy that keeps the top of the distribution — the best path pays +0.877 where `trail` closes the same token at +0.607. It gives back a little win rate (53% vs 55%): that is the trade, more per runner for slightly more often ending at the give-back stop. On the halves it is ahead of `trail` on the newer one (+0.01672 vs +0.01120) and a hair behind on the older (+0.04264 vs +0.04395).

Sweeps behind the two numbers that matter (`--rungs ... --giveback ... --gb-arm ...`):

- **Arming the give-back stop at the cost-back multiple, not at the first rung**, is worth +0.0009 a trade and most of the median at the shipped width. Half the sample peaks under 2x; with the stop armed only at 2x those moonbags had nothing under them at all.
- **Width has an optimum near 20-25%**: 20% +0.02998, 25% +0.02962, 40% +0.02770, 60% +0.02520 (below the fixed ladder). Wider rides reversals all the way down; tighter starts cutting the runs the rungs exist to hold. 25% is the middle of the plateau.
- **The ladder's back-loading is worth ~1-2%**: 10/20/30/40 beats 25/25/25/25 at every width tested (+0.02962 vs +0.02908 at 25%).

- **Max hold skip.** `MAX_HOLD_ACTION=skip` (or `LAUNCH_MAX_HOLD_ACTION` / `WHITELIST_MAX_HOLD_ACTION`, unset = global) drops a position at max hold instead of selling it.
  - The position stops being tracked and frees its slot. The bag is kept, and its cost basis is written off as a loss (`exit_reason=max_hold_skip`). What a sale would have returned is stored as the bag's value at skip.
  - Live, the tokens stay in the wallet with no stop-loss, and the write-off counts toward the daily-loss limit. A Telegram alert names the bag. `/sellall` still sells a position that hasn't been skipped yet.
  - For `SKIP_WATCH_HOURS` (default 6, 0 = off), a read-only watcher re-prices each bag every `SKIP_WATCH_POLL_SECONDS` (default 30). It records the last and peak value. `npm run report` lists the bags and compares selling at max hold with their last and peak value.
- With one position slot, the confirm entries (about a dozen an hour) would crowd out the rare bundle entries. Raise `MAX_CONCURRENT_POSITIONS` for the side-by-side paper run.

The filters run between the launch listener and the traders. A launch they reject never takes a position slot, so it can't block the launches right behind it; launches arrive several per second.

OG and creator are local SQLite queries. They run once the index has caught up to the launch's block, which takes one on-demand tail. The dev-holding read (two RPC calls) is made only for a launch that passed both. The whole check is bounded by `LAUNCH_FILTER_MAX_WAIT_MS` (default 4000); a lookup still running after that counts as unresolved.

A launch that passes prints a `PASS` line and goes on to the snipe-tax wait and the buy. Its `BUY` line carries `og=… creator=… dev=…`.

The rest are counted, not printed. Every ~30 s a `FILTER` line shows `N launches in 30 s · passed=… · name_seen 81 · symbol_seen 22 · …`. A `SKIP` line is printed only for a launch that got past OG and creator. Individual launches are not printed; `LOG_FORMAT=json` still writes a `launch_observed` record for each.

In the dashboard, a **launches → entry** panel under the open positions replaces the FILTER line. It shows:
- the launch rate by quote asset
- detection and decision latency
- a funnel of the last 5 minutes (seen → quote ok → OG → fresh creator → dev holding → snipe tax → entered) with the top reasons at each stage
- the newest launches, one row each, with their OG / creator / dev results and where they stopped. A launch waiting on the snipe tax shows the live tax. Click a row to open it on gmgn.

### Deployer whitelist (optional, launch entry only)

`LAUNCH_FILTER_DEPLOYER_WHITELIST=true` makes the launch entry buy **only** launches by the wallets in `LAUNCH_DEPLOYER_WHITELIST` (comma-separated). It is exclusive: whitelisted launches skip the launch filters above, and every other launch is skipped (`not_whitelisted`). The filters' settings stay in `.env` and apply again once the whitelist is off; startup warns that they are ignored while it is on.

- **Matching.** A wallet matches as the `TokenLaunched` deployer, or as the launch tx's **sender**. Serial launchers often relay through a contract that is a fresh deployer on every launch, so only the sender stays the same. The sender costs one `eth_getTransaction` per launch (cached, bounded by `LAUNCH_FILTER_MAX_WAIT_MS`). A sender it can't read is `whitelist_unresolved`, and the launch is skipped.
- **Entry.** With `WHITELIST_CONFIRM_WAIT_SECONDS=0` (the default), a match is bought as soon as the snipe tax allows, like any launch entry.
  - Set it to N (max 120) for a **momentum confirm**. The curve price when the tax first passes is the reference. At launch block time + N s the launch is bought only if the price is up at least `WHITELIST_CONFIRM_MIN_RISE_PCT` % (default 50); otherwise it is skipped as `no_momentum`.
  - A curve that closes before the check is skipped as `graduated_before_confirm`. The wait holds no position slot.
- **Size.** `WHITELIST_QUOTE_IN_WEI` (unset = `PAPER_QUOTE_IN_WEI`). Live whitelist buys use the smaller of it and the normal live shot, so it can only shrink them.
- **Exits.** `WHITELIST_TAKE_PROFIT_MULTIPLE`, `WHITELIST_SECOND_TAKE_PROFIT_MULTIPLE`, `WHITELIST_TAKE_PROFIT_SELL_PCT`, `WHITELIST_STOP_LOSS_MULTIPLE` (0 = no stop-loss) and `WHITELIST_MAX_HOLD_MINUTES` (max 1440). Each unset one falls back to the global rule; an unset second TP never falls below the first.
  - Live with the stop-loss off, `WHITELIST_MAX_HOLD_MINUTES` must be set. It is then the only exit on the way down: the position holds a slot, and its unrealized loss doesn't trip the daily-loss breaker.
- **Safety at startup.** The flag must be exactly `true` or `false`. Listing a shared contract (Multicall3, the launch router, the factory, or a known relayer) stops startup, because every launch through it would match. A filled list with the flag off prints a warning.
  - Trades are tagged `strategy=whitelist` (paper_trades and live_trades), and a restart keeps their exits.
- **Quotes.** Non-ETH launches are paper only (`LAUNCH_STOCK_QUOTES`), and only for quotes with a Chainlink feed in `chainlinkQuoteFeeds`. Live whitelist buys stay native-ETH only.

Output: a match prints `PASS … whitelist=sender 0x85c5…0ed8`, then (with the confirm on) `PASS … momentum=+62% at 20s` or `SKIP … reasons=no_momentum`. The `FILTER` summary counts `not_whitelisted`. On the dashboard, the funnel shows whitelisted → quote ok → tax → momentum → entered.

Backtest (2026-09-14) of the launcher that prompted this, `0x85c5…0ed8`: 196 launches in 9 days, 97% of them non-ETH.
- Buying at +2 s lost money under every TP/SL/hold tried.
- A 20 s / +50% confirm with TP 2× and no SL was +34% per trade over 25 trades.
- That setting was picked from many tried, so paper-trade both modes before any live money.

### OG research table

With `LAUNCH_FILTER_OG=true`, every launch the OG check registers as new goes into `og_launches`, whether or not it passed the other filters. Each row holds two kinds of data.

What was known at launch:
- name, ticker, quote asset
- socials, description and logo
- creator history
- the launch-tx dev buy (quote and ETH) and % of supply
- dev holding
- the filter verdict

What happened next:
- price and fill a few seconds in ("early", about when the bot buys)
- the peak over the first 5 minutes, polled every 5 s
- 1, 5 and 30 minute checkpoints
- when the curve sold out

Every tracked launch is read in one Multicall3 call per tick. `npm run research` splits the outcomes by each of those facts, so you can see which ones separate the runners from the trash:
- `npm run research -- --hours 6` limits it to the last 6 hours
- `npm run research -- --csv` writes every row to `./data/og-launches.csv`

Beyond the launch facts, it splits by dev wallets in the launch tx (1 / 2–3 / 4+), by whether the dev had sold by the early read (ETH), and by the paper strategy the bot bought under.

### Bundle-router lane (`ROUTER_BUNDLE_ENTRY`)

Some launch contracts create the token and buy it for several wallets in the same transaction. `0x3f50…0722` is the Pons version of Infinite Bundler: it is the `TokenLaunched` deployer, and it buys with 4–17 wallets inside the launch tx. You can't get in alongside that bundle. The research on its first 240 launches (2026-09-08 to 09-18) found two things instead:
- **Operator's first launch only.** The operator is the launch tx sender. Tokens from operators who had launched through the contract before graduated 0 of 70 times.
- **Follow the bundle's first-minute selling.** The bundle sells straight into the curve: the first sell comes at about 20 s, and half the bag is gone by about 46 s. So the entry waits for the moment the bundle is still holding while the curve keeps filling.
  - Setup B: the first moment 30–180 s after launch when the curve's ETH is up ≥ 0.6 from its post-bundle level and the bundle has sold ≤ 30% of its bag.
  - Result: 44 trades, +183% mean, +19% median, 55% win rate. The thresholds were picked from a grid, so paper-trade this before trusting it.

`ROUTER_BUNDLE_ENTRY=true` runs it on its own, independent of the other entry paths:
- **Detection:** its own factory poll, one `getLogs` for `TokenLaunched` with deployer ∈ `ROUTER_BUNDLE_ADDRESSES`.
- **Operator history:** the table `router_launches`.
  - At start it is backfilled from the factory's deploy block on `CREATOR_SCAN_RPC_URL`. Set that URL: a 10-block `getLogs` RPC can't do this.
  - Until the backfill is done, launches skip as `operator_history_incomplete`.
- **Watch:** every `ROUTER_BUNDLE_POLL_MS`, one Multicall3 call reads the curve's `realQuoteReserve` and each bundle wallet's balance.
  - The bundle wallets are the operator plus every launch-tx buy recipient.
  - The baseline is read at the launch block.
- **Skip reasons:**
  - `repeat_operator`
  - `non_native_pair`
  - `no_bundle_wallets`
  - `no_signal` (window over)
  - `graduated_before_signal` (v1 enters on the curve only; that was 1 of 87 entries in the research)
  - `stale`
- **Exits** (the research's), independent of `EXIT_STRATEGY`, always selling the whole bag:
  - −30% on the curve while the peak is under 1.3x
  - a 60% trailing stop on the curve once the peak reaches 1.2x
  - a 40% trailing stop in the v4 pool
  - a 6 h max hold

Live (`LIVE_TRADING=true` with `ROUTER_BUNDLE_ENTRY=true`, and no other entry path), the lane runs through the same executor as a curve-close shot — claim, snapshot, rate-bounded buy, receipt validation, polled exits — with three differences: the shot is `ROUTER_BUNDLE_QUOTE_IN_WEI`, still capped by `BUDGET_PER_SHOT_WEI` and `LIVE_MAX_TRADE_WEI` and still counted against the daily and exposure caps; the exits above replace `EXIT_STRATEGY`; and there is no snipe-tax wait (the signal comes tens of seconds after launch, and the tax is re-checked on the snapshot the buy is built from).

Four things config refuses live, each because it would otherwise cost money quietly:
- no `CREATOR_SCAN_RPC_URL` — the operator history could never complete, so the lane would enter nothing;
- `ROUTER_BUNDLE_FIRST_LAUNCH_ONLY=false` — the wallet would buy repeat operators, which is not the setup that was backtested;
- `ROUTER_BUNDLE_STOP_LOSS_MULTIPLE=0` — a falling position would have no exit before the (up to 7-day) max hold;
- `ROUTER_BUNDLE_CURVE_TRAIL_ARM` above `ROUTER_BUNDLE_STOP_LOSS_UNTIL_PEAK` — a peak landing between them would have neither the stop nor the trail.

A launch whose factory record was unreadable at launch time (null `poolFee` / `tickSpacing`) is re-read once and refused if it is still missing: without the pool key a graduated position could be neither priced nor sold, so no exit could ever fire.

#### Setup C (`ROUTER_BUNDLE_SETUP=C`)

Setup C replaces setup B's watch; the two never run together. Setup B stays the default.
- **Gate:** a launch whose bundle (the operator plus the launch-tx buy recipients) has at least `ROUTER_BUNDLE_MIN_WALLETS` wallets (default 7).
- **Entry:** `ROUTER_BUNDLE_C_ENTRY_DELAY_SECONDS` after launch (default 45), and only if the curve price is then at least `ROUTER_BUNDLE_C_MIN_PRICE_RATIO` (default 0.8) × its post-bundle price. The check runs once, at the first watch sample at or past the delay. Price is (virtual quote / virtual quote at the launch block)², from `getReserves()`, which is exact on the constant-product curve. A delay of 0 restores the old immediate entry, and a ratio of 0 turns the price check off.
- **Why:** the synchronized-dump rug. The bundle pushes the price to about 4×, then every bundle wallet sells its whole bag in one block 10–50 s after launch and sends the proceeds to the disperse funder. That happened on 47 of 351 launches, and 14 of them passed the gate (for example ZEST and PERIWINKLE on Sep 23). No funder, hub or bundle-size filter separates them from normal launches. The +45 s / 0.8× rule took 2 of those rugs instead of 14: +1.32 vs +1.07 ETH over Sep 8–25 at 0.05 ETH with the pyramid, and +0.33 vs −0.02 since Sep 20. Research: wallet-tracker `analysis/ponsv2-bundler-0x52f0/`.
- **Snipe tax:** the paper and live traders then wait until `currentSnipeTaxBps` is ≤ `MAX_SNIPE_TAX_BPS`, polling every `LAUNCH_SNIPE_TAX_POLL_MS` and giving up after `LAUNCH_SNIPE_TAX_MAX_WAIT_MS`.
- **Unchanged from B:** the first-launch-only check, the size and the max hold.
- **New skip reasons:**
  - `bundle_too_small`.
  - `too_late`, for a launch first seen more than `ROUTER_BUNDLE_C_MAX_AGE_SECONDS` (default 10) after launch (e.g. a poll catching up after an outage), or when the delayed check's first sample lands more than that past the delay.
  - `dumped`, when the price is under the ratio at the check.
  - `no_baseline_price`, when the launch-block read failed (fail closed).
  - A launch waiting for its check reads `c_waiting`.
- **Exits:** the pyramid ladder (`EXIT_*` / `EXIT_PYRAMID_*`, whatever `EXIT_STRATEGY` says) with `ROUTER_BUNDLE_MAX_HOLD_MINUTES`. The `ROUTER_BUNDLE_STOP_LOSS_*` / `*_TRAIL_*` settings are unused. The bundle wallets' staggered dump in the first minute swings a launch entry well past arm and back (USDC 2026-09-22: 1.32x → 1.01x at +28 s, then 3.5x), which is why the pyramid has no break-even floor; live needs `EXIT_RUG_STOP_MULTIPLE` > 0.
- **Live:** config refuses it when both `EXIT_RUG_STOP_MULTIPLE` and `EXIT_NO_PROGRESS_MINUTES` are 0, because the pyramid would then have no downside exit before cost back. It also refuses `MAX_SNIPE_TAX_BPS` above 1000, because the tax is paid in full. Live, both setups' exit guards always run, because a restart under the other setup still resumes the open positions on their own exits.
- **Data:** positions are tagged `router_bundle_c`, and the watch still records each launch's path to `ROUTER_BUNDLE_MAX_AGE_SECONDS`.
- **Status:** the delayed entry was backtested in the research (Sep 20+ is only 55 trades, and most rugs fall on Sep 23). Run it on paper first.

Positions are tagged `router_bundle`, and `npm run report` shows each strategy's P&L on its own. Each watched launch stores its path (`ageMs:riseMilliEth:soldPpm`) to the end of the window, whether it signalled or not, so the thresholds can be re-tuned from our own data. Paper gas counts the buy, two approvals and the sale. At the default 0.0025 ETH size, that is a noticeable share of each trade.

## Reports

The CLI report follows the same operator pattern used by Freqtrade and Hummingbot: realized P&L is calculated from persisted exit legs, while open positions are shown separately and are not counted as realized profit.

```sh
npm run report
npm run report -- --json
npm run report -- --csv ./data/paper-trades.csv
```

The report shows entered launches, observed graduations, open and closed positions, exit count, win rate, realized P&L in wei and ETH, and committed quote. The CSV contains one row per paper trade and includes remaining tokens and exit reason.

For unattended operation after building:

```sh
pm2 start ecosystem.config.cjs
pm2 save
```

### What gets written

On the curve path, only tokens the bot entered: the `launches` row is written by the trader at the moment an entry is opened. Every graduate is written to `graduations`. Together with `paper_trades`, `paper_trade_exits` and `live_entries` (the buy/sell record), that is the whole audit trail. A launch or graduate that is skipped is logged to the console and not persisted. Telegram controls and live execution remain later phases in the brief.

### Graduation entry (paper only)

With `GRADUATION_ENTRY=true` a graduation watcher polls the factory for `PoolGraduated` (bonding curve complete, liquidity migrated into the Uniswap v4 pool). A graduate is entered as soon as it is detected, provided its block is within `MAX_EVENT_AGE_MS`, its pair is native ETH, its pool key is known, a position slot is free, and it passes the on-chain v4 quote and the thin-pool guard. There is no token-quality filter. A graduate that is skipped is logged with its reason (`non_native_pair`, `missing_pool_key`, `concurrency_limit`, `no_pool_price`, `quoter_unavailable`, `entry_slippage`, …).

The size is `GRADUATION_PAPER_QUOTE_IN_WEI`. The thin-pool guard is size-aware. The entry's price impact is measured by quoting the real size and a 1/1000 reference together, so the Pons fee cancels out of the ratio. The entry may use what the token's freshly seeded pool would charge at that size, plus `GRADUATION_MAX_SLIPPAGE_BPS`, and never more than `GRADUATION_MAX_IMPACT_BPS` (1000) outright. The fresh-pool charge is `size / (seed + size)`, which is about 230 bps at 0.1 ETH and 670 bps at 0.3 ETH on the usual 4.2 ETH seed. The guard rejects pools drained below their seed, not healthy pools that a larger size naturally moves. If the quoter is unreachable the entry is skipped (`quoter_unavailable`) rather than taken at spot.

Graduation positions exit on the same rules as curve positions: take-profit, second take-profit, stop-loss and max-hold, priced at the v4 pool's spot.

This is paper-only. The live executor never trades graduates, and `LIVE_TRADING=true` refuses to boot with `GRADUATION_ENTRY` on. Watcher tuning: `GRADUATION_POLL_INTERVAL_MS`, `GRADUATION_SCAN_BATCH`, `GRADUATION_MAX_TRACKED`, `GRADUATION_WATCH_TTL_MINUTES`.

### Curve-close entry (paper; live in Phase 3)

With `CURVE_CLOSE_ENTRY=true` a curve tracker follows every live Pons bonding curve. It keeps, per token, the quote collected (`realQuoteReserve`) and the `graduationThreshold`, so `remaining = threshold − collected` is always known. A curve fires once it reaches the trigger, `CURVE_CLOSE_TRIGGER_PCT_LEFT`: the percent of its graduation threshold still left, net of fees. `5` fires at 95% filled. That is ≤ 0.21 of 4.2 ETH, ≤ 1.21 of 24.2 GOOGL, or ≤ 404.5 of 8090 USDG. Each curve is measured in its own quote units, so the ETH value of a stock or USDG curve never moves the trigger. Decimals are fine (`2.5`).

The trigger and the shot size are separate. `BUDGET_PER_SHOT_WEI` is what one entry spends; the trigger is how full the curve must be. With `BUDGET_PER_SHOT_WEI=100000000000000000` and `CURVE_CLOSE_TRIGGER_PCT_LEFT=12`, the bot buys 0.1 ETH once a curve is 88% filled. On a 4.2 ETH curve 0.5 ETH is still left then, so that `BUY` line reads `short-of-close`.

With `CURVE_CLOSE_TRIGGER_PCT_LEFT` unset, the bot only enters when one shot can close the curve: the ETH value of what's left must be at most `BUDGET_PER_SHOT_WEI`. The old `CURVE_CLOSE_TRIGGER_WEI` is gone. If it's still set, the bot refuses to start and names the replacement.

- **Source.** The tracker reads `CurveBuy` / `CurveSell` events from every contract. Curve trades go straight to each launch's curve; they never pass through the Meme hook, which only sees v4 swaps after graduation. The factory has to confirm an emitter as the curve of a live phase-0 launch before it is tracked, so a look-alike event from another contract is ignored. A buy adds `quoteIn − fee − tax` to `collected`, and a sell removes `quoteOut + fee + tax`. Both formulas were checked to the wei against on-chain reads.
- **Thresholds.** `CURVE_TRACK_THRESHOLDS` defaults to `4.2,24.2,41.6,10.9,72.2`. Only 4.2 is native ETH. The rest are Robinhood stock-token quotes (24.2 AAPL/GOOGL, 41.6 NVDA, 10.9 SPY, 72.2 SPCX). They are tracked and shown in their own units, but never fired.
- **Fire.** A `CLOSE` line prints when a curve enters the zone. The paper trader then re-reads the curve live and re-checks the gate. A curve that is already closed is skipped. One pushed back over the trigger is still entered, unless `CURVE_CLOSE_REQUIRE_IN_BUDGET=true` (then it is skipped as `remaining_above_trigger`). The trader also applies `MAX_SNIPE_TAX_BPS` and the position slot, then quotes a `BUDGET_PER_SHOT_WEI` buy in the curve's own integer math. The fill is clamped to the last sellable tokens and the excess is refunded, so the cost basis is what was actually spent. The `BUY` line shows `closes` or `short-of-close`: the gate is net, so a budget within 1–3% of remaining may not cover the fees. Each token is entered once (`paper_trades.strategy = 'curve_close'`), and it exits on the usual TP/SL/max-hold rules.
- **Paper limitation.** Live, a shot at least as big as what's left would itself finish the curve and graduate the token. On paper nothing moves on chain, so the position is priced on the curve until someone else closes it, then at v4 spot.
- **Per-block view.** Curves are evaluated at every block boundary, because a curve often sits within budget for only 0.1–0.3 s (1–3 blocks) before someone closes it. When a tracked curve closes, a `DONE` line shows how much was left when it was last seen, how many blocks earlier, and whether it was in the trigger zone. The tracker polls every `CURVE_TRACKER_POLL_INTERVAL_MS` (default 250), with feed nudges in between, against a head no older than `CURVE_TRACKER_HEAD_CACHE_MS` (default 200).
- **Sequencer feed** (`SEQUENCER_FEED_URL=wss://feed.mainnet.chain.robinhood.com`, optional). The feed streams every transaction about 150 ms before the RPC has its receipt. When a transaction goes straight to a curve that is inside the trigger or the prefetch zone, the tracker polls that receipt right away and applies its real `CurveBuy`/`CurveSell` log, without waiting for the next `getLogs` window. The later `getLogs` copy of that log is skipped by block and log index, so nothing is counted twice. A `CLOSE` line fired this way ends in `(feed)`, and the panel shows `feed: N txs followed · M trades early`. About 60% of curve trades go through routers or smart accounts. Their target curve isn't visible in the transaction, so they still arrive through `getLogs`.
- **Feed-triggered position reads** (`FEED_POSITION_TRIGGER`, on by default when the feed is set). A position's price only moves when someone trades its token, so exits don't wait for the `POSITION_POLL_INTERVAL_MS` timer. A feed transaction that names an open position's curve or token anywhere in its bytes schedules one snapshot read of that position. This covers direct curve trades, routers, the v4 pool after graduation, and 4337/7702 calls. The read runs `FEED_POSITION_READ_DELAY_MS` (200 ms) later, which is when the transaction has normally been mined. Live and paper both work this way. There is no receipt polling, so each trigger costs one `eth_call`. A burst of trades collapses to at most one read per `FEED_POSITION_MIN_INTERVAL_MS` (250 ms), and the last read always comes after the last trade. A rate-limited read (429) pauses that position's triggers for `FEED_POSITION_BACKOFF_MS`. The timer stays as the safety net: it skips a tick when a triggered read just ran, and it runs every `POSITION_POLL_INTERVAL_FEEDLESS_MS` (1 s) while the feed is disconnected. RPC calls therefore follow trades on the tokens you hold, not the clock: a quiet position costs the timer only, a busy one up to 4 reads a second while it is being traded. Without `SEQUENCER_FEED_URL`, positions poll on the timer exactly as before.
- **Monitoring.** Every `CURVE_TRACKER_SUMMARY_MS` a summary line prints the curves tracked per quote asset, the closest-to-close curves, the cursor lag behind head in blocks, and RPC health: getLogs count, average latency and errors, head-not-ready retries, admissions, resyncs, and the node's last error text. With the split-screen dashboard on, this goes to a **curves → close** panel under the open positions instead of the event log. The panel refreshes every second and lists the curves closest to the trigger, across all quote assets, ranked by fill. Each row shows a fill bar, what's left in the quote's own units and in ETH, the token's age, and the time since its last trade. `▶` marks curves inside the trigger and `›` curves inside the prefetch zone; the ETH amount is green when one shot would close the curve. Click a row to open the token on GMGN. Headless runs (PM2, pipes, `LOG_FORMAT=json`) still print the line. A tracker more than `CURVE_TRACKER_MAX_LAG_BLOCKS` behind head (default 600, about 60 s) drops its state and restarts at head rather than trigger on old data. Tuning: `CURVE_TRACKER_POLL_INTERVAL_MS`, `CURVE_TRACKER_MAX_TRACKED`, and `CURVE_TRACKER_FETCH_CONCURRENCY` (parallel getLogs windows while catching up).
- **Stock- and USDG-quoted curves** (`CURVE_CLOSE_STOCK_QUOTES=true`). Curves quoted in Robinhood stock tokens (24.2 AAPL/GOOGL, 41.6 NVDA, 10.9 SPY, 72.2 SPCX) and in the USDG stablecoin (threshold `8090` USDG, 6 decimals) are traded too. Thresholds are listed in each asset's own units and matched at its own decimals; curves whose threshold isn't listed are counted in the summary line as `untracked:`.
  - What's left on the curve is valued in ETH from the Chainlink feeds on Robinhood Chain (stock/USD and ETH/USD; a stock feed already includes the token multiplier), and a curve without a fresh price is never entered. With `CURVE_CLOSE_TRIGGER_PCT_LEFT` set, the gate is the share of the threshold left, not that ETH value.
  - The paper shot swaps the ETH budget into the stock at the oracle price, paying `CURVE_CLOSE_QUOTE_SWAP_COST_BPS` (default 30). It buys the curve in stock units, and on exit converts the proceeds back the same way. P&L, the report and the dashboard are all in ETH.
  - The equity feeds run 24/5. A price older than `CURVE_CLOSE_QUOTE_PRICE_MAX_AGE_HOURS` (default 72, which spans a weekend) is skipped with `no_quote_price`.
  - With the dashboard on, the feed prices show in a **stock-quote prices** panel under **curves → close** instead of a startup log line. The panel shows each asset's USD price, ETH per token and feed age, refreshed every minute. Ages over a day are yellow (normal on a weekend for the 24/5 equity feeds). A feed past `CURVE_CLOSE_QUOTE_PRICE_MAX_AGE_HOURS` is red and marked `stale: not traded`, and a stale ETH/USD feed marks every stock stale.
  - With the flag off, stock curves are tracked and shown but never traded.
- **Filters (optional, this path only).** Each filter has its own switch; with several on, a token must pass all of them. A skipped token prints `SKIP … creator=… age=… dev=… reasons=…`.
  - **A: creator history** (`CURVE_CLOSE_FILTER_CREATOR=true`). The creator is the `TokenLaunched` deployer, i.e. the wallet or contract that launched the token, even through the launch router. `CURVE_CLOSE_CREATOR_MODE` picks the rule:
    - `fresh`: the shot is taken **only on the creator's first token**. Any earlier launch by the same deployer skips it (`creator_not_fresh`), and so does an unknown history. Serial launchers (a wallet with dozens or thousands of launches) never pass.
    - `repeat` (default): the shot is taken only when the creator had at least `CURVE_CLOSE_MIN_CREATOR_PRIOR_LAUNCHES` launches before this token. That excludes fresh creators.
    - The count comes from the **launch index** (below): one local query, exact once the index is complete. The `creator=` field shows the count and the creator's last 3 prior symbols, e.g. `repeat(4: PEPE,DOGE,CAT)`, or `repeat(>=2)` while the backfill is still running.
    - A lookup the index can't answer yet is unresolved (`creator=unresolved(<reason>)`) and the token is skipped. Set `CURVE_CLOSE_UNRESOLVED_CREATOR=trade` to take it anyway. Reasons: `index_backfilling_<pct>pct` (first run, the creator has too few priors found so far; in fresh mode, "none found yet" isn't proof of a first token), `not_indexed_yet` (launched after the last poll and not found on an on-demand poll either), `shared_contract_deployer` (launched through a shared batcher such as Multicall3).
    - A trigger waits at most `CURVE_CLOSE_CREATOR_MAX_WAIT_MS` (default 1500) for a lookup that is still running, and prints a `WAIT` line while it does (`creator=pending` after that). Filter B is checked first, so a token B already rejects never waits.
  - **B: launch → migration time** (`CURVE_CLOSE_FILTER_LAUNCH_AGE=true`). The shot lands at migration, so the filter measures launch → now, using the curve's own `launchedAt()`. The token is skipped when that falls between `CURVE_CLOSE_EXCLUDE_AGE_MIN_MINUTES` and `CURVE_CLOSE_EXCLUDE_AGE_MAX_MINUTES` (defaults 5 and 120, both inclusive). If the launch time is unknown, the token is skipped. To choose the window from data, run **`npm run close-age`** (`src/research/close-age-study.ts`). It replays the curve-close shot on every graduation since `--since` (default: the 1st of the month), native and stock/USDG-quoted, the latter valued at historical Chainlink prices. Each shot is exited with the live `.env` exit, and the script prints P&L by launch→trigger age bucket plus a MIN/MAX sweep scored for robustness. It only sees curves that graduated; `--db` adds the paper trades it can't see.
  - **C: dev holding** (`CURVE_CLOSE_FILTER_DEV_HOLDING=true`). The token is skipped when its dev wallets still hold at least `CURVE_CLOSE_MAX_DEV_HOLDING_PCT` (default 20) % of the total supply as the curve closes: a dev sitting on a big bag as the token hits the AMM is the setup for the dump right after.
    - Dev wallets are the deployer, the launch tx's sender, and every recipient of a curve buy inside the launch tx. On this chain a launch-and-buy through the router sends the tokens to the sender, a launch through a smart contract can send them to a different EOA than the deployer, and a bundled launch buys for several wallets at once.
    - `dev=12.3% (bought 15%, kept 82%)`: what the wallets hold now, what the launch-tx buys took, and held ÷ bought. "Held" includes later buys by those wallets and excludes what they sold or moved on; tokens moved to wallets never seen in the launch tx are invisible.
    - The launch tx is read once per token; the balances are re-read every few seconds while the curve is within `CURVE_CLOSE_FILTER_PREFETCH_WEI` of closing, so the trigger normally uses a read under 10 s old without waiting. Unknown holdings are skipped.
- **Launch index** (`src/launch-index.ts`). It runs when curve-close A or C is on, or when the launch OG or fresh-creator filter is on. It stores every `TokenLaunched` since `FACTORY_DEPLOY_BLOCK` (token, deployer, block, launch tx, name, symbol) in the bot's SQLite database, table `launch_index`. Name and symbol are also stored as keys for the OG check.
  - The first run backfills the history once in the background, newest first, so recent creators resolve within the first requests. On the public RPC this is ~350k launches, a few minutes. After that it polls every `LAUNCH_INDEX_POLL_MS` (default 2000), and a restart only fetches the blocks it missed. A token launched since the last poll is fetched on demand.
  - Near head the index doesn't use `CREATOR_SCAN_RPC_URL`.
    - When the launch listener runs (`PRE_GRADUATION_ENTRY`), the index takes every window of launches the factory poller already fetches, so it makes no requests of its own.
    - Otherwise, in curve-close-only runs, it polls the main RPC in `GET_LOGS_MAX_RANGE` windows.
  - `CREATOR_SCAN_RPC_URL` only backfills history and closes long gaps, such as the blocks missed while the bot was stopped, in one wide request.
  - When that request fails (an error, or 3 throttled answers), the main RPC closes the gap 10 windows per poll. The scan RPC is tried again 30 s later.
  - On 2026-09-15 the public RH RPC 429'd even a steady 0.5 req/s, and its backend timed out at times ("context deadline exceeded", which viem reports as "Missing or invalid parameters").
  - Its getLogs run on `CREATOR_SCAN_RPC_URL`. Providers that cap getLogs at a few blocks per request can't backfill (startup warns), so set it to `https://rpc.mainnet.chain.robinhood.com`: that RPC returns up to 10,000 logs per request and rate-limits bursts, and the index paces itself, halves its span on a "too many logs" answer, and backs off on 429s.
  - `LAUNCH_INDEX_NAMES` (default true) fills each token's name and symbol, newest first, in paced Multicall3 reads on the main RPC. It never blocks the index or a lookup.
- **Baselines.** A curve's starting value comes from one eth_call at `latest`: Multicall3 bundles `ArbSys.arbBlockNumber()`, `token()` and `realQuoteReserve()`, so the value and its L2 block are exact. It never reads historical state, which pruned or load-balanced RPCs reject with "Missing or invalid parameters".

## Phase 3: live curve-close trading

`LIVE_TRADING=true` trades the curve-close path with real money, using the same trigger, A/B/C filters and exit rules as paper, and exits automatically. It is locked by default, and `LIVE_DRY_RUN` is **on** unless you set it to `false`, so turning live on never broadcasts by accident. Only the curve-close path trades live. The bot refuses to boot with `LIVE_TRADING=true` unless `CURVE_CLOSE_ENTRY=true`, `PRE_GRADUATION_ENTRY=false` and `GRADUATION_ENTRY=false`. The live launch code is kept, but can't be reached while that guard stands.

**Every transaction** follows the same path: fresh quote from one atomic snapshot → a minimum-output bound → simulation (`eth_estimateGas`) → signed locally and **written to the journal before broadcast** → sent → validated against the receipt and on-chain balances. What gets recorded is the receipt and the balances, never the quote. The execution facts this relies on were verified against the live contracts on chain 4663 with state-override simulations:
- the buy and sell math match the curve to the wei
- a buy's `minTokensOut` is a *rate* check, so a buy clamped at sellout still passes
- sells need an approval
- the real revert selectors (`SlippageExceeded(uint256,uint256)`, not the docs' parameterless form)
- the refund semantics
- the UniversalRouter calldata, byte-identical to a real Pons sale

**Entries.**
- **Once per token:** each token can be entered once (a unique row, so a restart or a second trigger can't re-enter).
- **Caps reserved up front:** the daily spend cap, exposure cap and position slots are reserved with no await in between.
- **Checks on a fresh snapshot:** the launch record, not sold out or graduated, snipe and creator tax, and wallet ETH above shot + gas reserve.
- **Retries only when nothing executed:** a buy is retried only when the previous attempt provably didn't execute (a simulation revert, never broadcast, or a slippage revert). A buy with no receipt by the timeout is **never** resent as a new transaction; the same signed bytes are rebroadcast.
- **Validation after a fill:** the fill must match the token balance change, the rate bound, and the refund. A mismatch marks the position `UNVERIFIED`, and the balance is taken as the truth.
- **Exit prep:** right after the buy, the approvals for every exit venue are made and the bag's sellability is checked.

**Exits.** Positions are polled like paper and decided by the shared rules (`src/exit-rules.ts`). Amounts come from `balanceOf`, never the database.
- **Venues:** before graduation, curve `sell()`; after, a v4 sale through Uniswap's official UniversalRouter (Permit2). If a curve sells out without graduating (phase 1: unsellable), the bot calls the permissionless `createGraduatedPool` itself.
- **Take-profits:** a fixed slippage bound, split into chunks when price impact is too high.
- **Stop-loss / max hold / `/sellall`:** start at `LIVE_STOP_SLIPPAGE_BPS`, widen per failed attempt up to `LIVE_STOP_SLIPPAGE_MAX_BPS`, and keep retrying until flat.
- **One sale at a time** per position.
- **Fees and gas:** each trade records the fee, creator tax and snipe tax paid (from the curve events) and all gas, so PnL is net.

**Risk controls.**
- **Per-trade and exposure caps:** `LIVE_MAX_TRADE_WEI` per shot, and `LIVE_MAX_EXPOSURE_WEI` across open positions.
- **Daily cap:** `LIVE_DAILY_CAP_WEI` per UTC day, rebuilt from the journal after a restart.
- **Gas reserve:** `LIVE_GAS_RESERVE_WEI` is never spent on entries.
- **Fee cap:** `LIVE_MAX_TX_FEE_WEI` per transaction.
- **Circuit breakers:** entries pause after `LIVE_MAX_CONSECUTIVE_FAILURES` failed transactions in a row, when today's realized loss reaches `LIVE_DAILY_LOSS_LIMIT_WEI`, or on a low wallet balance.
- **Stale-data guard:** no shot while the tracker is more than `LIVE_MAX_TRACKER_LAG_BLOCKS` behind the chain.
- **Pausing:** the stop file `data/LIVE_STOP` and Telegram `/pause` stop **entries only**; exits always run.

**Telegram** (`TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_IDS`).
- **Position cards:** each buy posts one pinned card for that position, with 🔴 Sell / ⏭ Skip / 🔄 buttons. The card is edited in place on every sale and, silently, with the live PnL every `TELEGRAM_CARD_REFRESH_MS` (default 20 s, 0 = only on trades). It is unpinned when the position closes or is skipped. Its message ids are journalled, so a restart keeps editing the same card. In a group, the bot must be an admin to pin.
- **Alerts:** every sale (partial or final), with PnL in ETH and USD (Chainlink ETH/USD), plus failures, breaker trips and unverified fills.
- **Commands:** `/status`, `/positions`, `/pause`, `/resume` (also clears a breaker), and `/sellall confirm`.
- **Access:** only the listed chats are answered.
- **Paper mode:** with `LIVE_TRADING` off, the bot shows the paper trader instead: the same pinned cards and sell notifications, tagged 🧻 PAPER. Sell and `/sellall` close the simulated bag at the current price, and `/status` shows paper exposure and today's realized PnL. There is nothing to pause. In a live dry run, Telegram stays on the live executor.

**Stock/USDG curves** (`LIVE_STOCK_QUOTES=true`). A swap-then-buy would lose a close race that lasts 0.1–0.3 s, so the wallet holds a float of each quote token that you fund. A curve approaching the trigger is pre-approved for exactly one shot (+10%, never unlimited), so the shot is a single buy. Costs and proceeds are valued in ETH through the Chainlink feeds.

**Crash safety.** On boot, the journal is reconciled against the chain. A transaction the chain never saw is never rebroadcast, because it was decided on stale prices. A position whose buy landed but wasn't recorded is rebuilt from its receipt.

**Rollout.**
1. Rotate any API keys that have ever been pasted anywhere.
2. **Dry run** (`LIVE_TRADING=true`, `LIVE_DRY_RUN=true`, `PAPER_TRADE=true`) for 1–2 days, with the canary wallet already funded. Simulations run from it and a gas estimate needs balance, but a dry run spends nothing. Every trigger produces a simulated, signed, unsent transaction (`live_dry_run` lines with quote, bound, gas and latency), and paper shadows it.
3. **Canary:** a fresh wallet with ~0.03 ETH, `LIVE_MAX_TRADE_WEI=5000000000000000`, one position, `LIVE_DAILY_CAP_WEI=20000000000000000`, ETH curves only, Telegram on, `LIVE_DRY_RUN=false`, `PAPER_TRADE=false`. Check every transaction on the explorer. It passes when:
   - several round trips have fills that match the explorer to the wei
   - a take-profit chunk is followed by a full exit to zero
   - a `/sellall` works
   - a restart mid-position reconciles cleanly
4. Then `LIVE_STOCK_QUOTES=true` with a small float, and raise the caps step by step.

`npm run report` includes a LIVE section: trades, realized PnL after gas, fees and taxes per trade, transaction states, and unverified fills.
