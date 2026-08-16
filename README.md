# Polymarket BTC/ETH Up-or-Down — order-book price dataset

Free, openly shared tick-level **order-book snapshots** for Polymarket's short-term
**Bitcoin & Ethereum "Up or Down"** markets (the 5-minute and 15-minute windows).
Every few seconds a collector records the top-of-book for both outcome tokens of
each active window, so you can reconstruct what the market was pricing at any
moment and backtest strategies against **real, tradeable prices** — not just
mid-marks. Markets settle via Polymarket's official resolution (Chainlink TWAP).

**This repo updates every time a database reaches 200 MB.** The collector fills a
file to ~200 MB, then rolls over to a new one — at that moment the completed file
is sealed and published here. Only these complete files are included; the file
currently being written is held back until it too reaches 200 MB, so everything in
this repo is final and won't change.

## Coverage

|                      |                                   |
| -------------------- | --------------------------------- |
| **Start of data**    | **2026-08-06 10:43:07 UTC** |
| **End of data**      | **2026-08-16 02:44:59 UTC** |
| **Total snapshots**  | 3,243,156 rows |
| **Assets tracked**   | BTC, ETH |
| **Window lengths**   | 5-minute and 15-minute |
| **Last updated**     | 2026-08-16 02:48:41 UTC |

*Updated automatically every time a database fills to 200 MB and is sealed; the
dates above reflect the latest published file.*

## Download

All databases live in the [`databases/`](databases) folder of this repo,
compressed with zstd. Click a file below, or download directly:

| File | Period (UTC) | Rows | Raw | Compressed | Ratio |
| ---- | ------------ | ---- | --- | ---------- | ----- |
| [`pm_btc_eth_5m_15m_20260806T104307Z_0000.db.zst`](databases/pm_btc_eth_5m_15m_20260806T104307Z_0000.db.zst) | 2026-08-06 → 2026-08-08 | 805,308 | 200.5 MB | 23.7 MB | 8.4× |
| [`pm_btc_eth_5m_15m_20260808T190001Z_0001.db.zst`](databases/pm_btc_eth_5m_15m_20260808T190001Z_0001.db.zst) | 2026-08-08 → 2026-08-11 | 816,040 | 200.3 MB | 23.4 MB | 8.6× |
| [`pm_btc_eth_5m_15m_20260811T040001Z_0002.db.zst`](databases/pm_btc_eth_5m_15m_20260811T040001Z_0002.db.zst) | 2026-08-11 → 2026-08-13 | 810,056 | 200.6 MB | 24.0 MB | 8.3× |
| [`pm_btc_eth_5m_15m_20260813T123001Z_0003.db.zst`](databases/pm_btc_eth_5m_15m_20260813T123001Z_0003.db.zst) | 2026-08-13 → 2026-08-16 | 811,752 | 200.5 MB | 24.5 MB | 8.2× |

```bash
# download one file
curl -L -O https://github.com/gensx-x1/polymarket-btc-eth-updown-data/raw/main/databases/pm_btc_eth_5m_15m_20260806T104307Z_0000.db.zst

# decompress -> you get back the exact original SQLite database
zstd -d pm_btc_eth_5m_15m_20260806T104307Z_0000.db.zst

# open it
sqlite3 pm_btc_eth_5m_15m_20260806T104307Z_0000.db "SELECT COUNT(*) FROM ticks;"
```

## How the files are compressed (and why)

Each `.db` is compressed with **[zstd](https://github.com/facebook/zstd)** at level
`6` (`zstd -6`) before it is committed, producing the `.db.zst`
files in [`databases/`](databases).

- **Why compress at all.** A raw database is ~200 MB. SQLite files full of repetitive
  numeric ticks shrink dramatically — here about **8.4×** (≈200 MB → ~24 MB),
  so downloads are far faster and each file stays well under GitHub's 100 MB limit.
- **Why zstd specifically.** It gives near-gzip-or-better ratios while decompressing
  extremely fast (hundreds of MB/s), so unpacking takes a moment. Level `6`
  is the sweet spot — most of the ratio for a fraction of a second of CPU.
- **It is lossless.** `zstd -d` reconstructs the **byte-for-byte original** SQLite
  database — nothing is dropped or rounded. Verify with a checksum if you like.
- **Getting zstd:** `apt install zstd` (Debian/Ubuntu), `brew install zstd` (macOS),
  or download from the [zstd releases](https://github.com/facebook/zstd/releases).

```bash
# verify a decompressed file (optional)
zstd -d -c pm_btc_eth_5m_15m_20260806T104307Z_0000.db.zst | sha256sum
```

## How the data is structured

Each database is a single SQLite file with **one table, `ticks`**. A row is written
for every poll of every active market window (roughly every few seconds), capturing
the full top-of-book for **both** outcome tokens at that instant. To reconstruct a
window, filter by `slug` (or `window_start`) and order by `ts_ms`.

The two outcome tokens are **Up** and **Down**; a winning token settles to **\$1.00**
and the loser to **\$0.00**. `up_*` columns are the Up-token book, `down_*` the
Down-token book.

| Column | Type | Meaning |
| ------ | ---- | ------- |
| `ts_ms` | int | snapshot time, epoch **milliseconds** (UTC) |
| `iso` | text | same instant as ISO-8601, e.g. `2026-08-13T12:00:00.363Z` |
| `asset` | text | `BTC` or `ETH` |
| `slug` | text | Polymarket market slug, e.g. `btc-updown-15m-1786298400` |
| `window_label` | text | `5m` or `15m` |
| `window_s` | int | window length in seconds (300 or 900) |
| `window_start` | int | window open, epoch **seconds** |
| `window_end` | int | window close, epoch **seconds** |
| `secs_into` | real | seconds elapsed since the window opened |
| `secs_to_close` | real | seconds remaining until it closes |
| `up_bid` / `up_ask` | real | best bid / ask for the **Up** token (0–1, = probability) |
| `up_mid` | real | midpoint `(up_bid + up_ask) / 2` |
| `up_last` | real | last traded price of the Up token |
| `up_bid_sz` / `up_ask_sz` | real | size (shares) resting at best bid / ask |
| `up_spread` | real | `up_ask − up_bid` |
| `down_bid` / `down_ask` | real | best bid / ask for the **Down** token |
| `down_mid` | real | midpoint of the Down token |
| `down_last` | real | last traded price of the Down token |
| `down_bid_sz` / `down_ask_sz` | real | size at best bid / ask (Down) |
| `down_spread` | real | `down_ask − down_bid` |
| `ask_sum` | real | `up_ask + down_ask` (≈ 1 + the market's edge/fees) |
| `ok` | int | `1` if the snapshot is complete and usable, else `0` |

Notes & conventions:

- **Prices are probabilities in `[0, 1]`** — an `up_ask` of `0.73` means it costs
  \$0.73 to buy one Up share that pays \$1 if Up wins.
- **Up + Down are one market.** At any instant `up_bid ≈ 1 − down_ask` and
  `down_bid ≈ 1 − up_ask`; `ask_sum` slightly above `1` is the round-trip cost.
- Multiple windows are live at once (a 5m and a 15m per asset), so a single
  timestamp has several rows — always scope by `slug`.
- Filter `WHERE ok = 1` for clean analysis.
- Each `slug` maps to a live/expired market at `https://polymarket.com/event/<slug>`.
- Files roll over at ~200 MB; together they form one continuous series — dedupe on
  `(slug, ts_ms)` if you load several at once.

Example — the entry ask of the Up token 3 minutes into every 15-minute BTC window:

```sql
SELECT slug, up_ask, secs_into
FROM ticks
WHERE asset = 'BTC' AND window_label = '15m' AND ok = 1
  AND secs_into >= 180
GROUP BY slug
HAVING secs_into = MIN(secs_into);
```

## Support this project 💛

I collect and share this data **completely for free** — no paywall, no signup,
no strings. Running the collector 24/7 and keeping this dataset fresh does cost a
little in server time, so **if it's useful to you and you'd like to see it keep
going, donations are very welcome and keep it alive:**

- **Ethereum / EVM (ETH, USDC, …):** `0x8038d57653eE77688919cbd346A2BA89534f7940`
- **Bitcoin:** `bc1q47gyzxgrtmc3j6afqs323g9r02586m8kgr206z`
- **Solana:** `C7mAb7cye9z4xJLNYVWbFSuutPWonbAHq9BDxKm8Ha59`

Even a small tip means a lot and directly funds keeping the collector running.
Thank you! 🙏

## Disclaimer

Provided **as-is**, for research and educational use, with no warranty of
accuracy, completeness, or fitness for any purpose. This is **not** financial
advice. Trading prediction markets is risky. Nothing here is affiliated with or
endorsed by Polymarket.
