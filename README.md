# Remote Auto-Trader (Iranian Crypto Exchanges)

Working copy of a live crypto auto-trading bot that buys and sells coins on Iranian exchanges
(SwapWallet, BitPin) driven by market ingestion, technical analysis, and an LLM "AI gatekeeper"
that vetoes candidate trades before execution. This is a genuine automated execution system, not
a paper-only research notebook: the default path submits real orders. Several top-level modules
referenced by `main.py` are not present here, so the folder does not run as-is.

**Suggested repo name:** `remote-auto-trader`
**Stack:** Python 3 (`requests`, `sqlite3`, rotating-file logging), a Go sidecar (`trendfeed.go`),
Gemini/OpenAI-compatible LLMs, GMGN/AltFins/CoinStats/Binance/Twitter/Reddit/Apify data feeds
**Status:** active (incomplete local copy)
**Last modified:** 2026-09-15

## What it does

`main.py` runs the poll loop: ingest market and social data, analyze into candidate signals, then
hand them to the execution module.

- `ingestion.py` - pulls balances and prices from SwapWallet, analytics from BitPin, klines/tickers
  from Binance, signals from AltFins, metrics from CoinStats, plus sentiment; computes RSI, EMA,
  Bollinger, Hurst, FFT and BTC-correlation and a neuro-fuzzy composite score.
- `analysis.py` - turns the feed into buy signals; delegates to `sp2l_strategy` and
  `strategy_trend_pullback.evaluate_setup`.
- `execution.py` - `ExecutionModule` places orders over the SwapWallet v2 API and BitPin, tracks
  per-wallet cooldowns (30 min default), persists positions, and routes across a primary/fallback/
  tertiary wallet set. It holds the AI gatekeeper and consults it inside `_place_order`.
- `ai_gatekeeper.py` - the pre-trade decision wall. Scores a candidate against multi-timeframe
  technicals, candlestick sequences, GMGN smart-money inflows and sentiment via Gemini (with a
  rotating multi-key pool and cooldowns, and an OpenAI-compatible backup relay). It rejects buys
  that look like bull traps, exhausted pumps or honeypots using `min_confidence_score` /
  `max_trap_risk_score`, and has a configurable `fail_safe_action` when the LLM is unreachable.
  In the committed config the fail-safe is set to `APPROVE`.
- `regular_paper.py` - a separate, explicitly non-live reference-market simulator for
  signal/exit comparison (never signs real orders), fed by `regular_market_data.py`.
- `trendfeed.go` - high-throughput Binance kline fetcher emitting the exact JSON shape
  `regular_market_data` produces, used to warm the trend-pullback strategy faster.
- `karlancer_scraper.py` - a HTTP-only Karlancer.com freelance-project scraper (unrelated to
  trading); it feeds a separate mediabot poller and landed in this folder by accident.

## Layout

```
main.py                    entrypoint: ingestion -> analysis -> execution loop
config.json                ALL provider + exchange secrets live here
ingestion.py               market + social data fetch, indicator/score computation
analysis.py                signal generation (sp2l, trend-pullback)
execution.py               ExecutionModule: SwapWallet/BitPin orders, positions, wallets
ai_gatekeeper.py           LLM pre-trade veto wall with multi-key rotation
regular_paper.py           paper-only reference simulation
regular_market_data.py     candle/quote provider (mirrors trendfeed output)
strategy_trend_pullback.py trend-pullback-v1 setup evaluator
trendfeed.go               Go Binance kline fetcher
```

## Notes

- **Missing modules.** `main.py` imports `gmgn_wallet`, `gmgn_engine` and `telegram_scraper`;
  `execution.py` imports `bitpin_client`; `analysis.py` imports `sp2l_strategy`. None are present
  here - this folder is a partial snapshot of a larger deployed bot (a Linux `/root/auto_trader`
  tree is referenced by `apply_seekai.py`). It will not import, let alone run, standalone.
- **Secrets.** `config.json` holds live exchange bearer tokens, X/Telegram/Reddit/Apify keys, a
  Gemini key pool and LLM relay keys. `seekai_test.py` and `apply_seekai.py` hard-code additional
  API keys in source. Treat every credential in this folder as exposed and rotate before publishing;
  do not commit `config.json`.
- **Real-money default.** `trading_settings.dry_run` is `false` and `degen_gmgn_live` is `true` in
  the committed config, and the gatekeeper's fail-safe is currently `APPROVE` - an LLM outage would
  let trades through. Set `dry_run: true` and `fail_safe_action: REJECT` before running against a
  funded wallet.
- **Debris.** `ai_gatekeeper_cur.py` is a byte-identical duplicate of `ai_gatekeeper.py`;
  `snap0910.tar.gz` is a dated snapshot of the core modules; the `patch_*.py` / `fix_explainer.py`
  scripts and `keta_logo_*.png` assets belong to other projects; `__pycache__/`, the `trendfeed`
  binary and `trendfeed_test.exe` are build artifacts.
- `trendfeed_test.exe` (~9 MB) and the compiled `trendfeed` binary are committed; gitignore them.
