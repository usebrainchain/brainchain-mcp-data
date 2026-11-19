# BrainChain MCP data

A daily record of Robinhood Chain tokens: every dated public step each one has taken, and where it went on to trade. Use it to test an agent against what was public on a given date. Each day's files are fixed by a hash written to BrainChain, so anyone can check that a day's files are the ones published that day.

The files are built by `brainchain-mcp dataset`, a command in [the BrainChain MCP](https://github.com/usebrainchain/brainchain-mcp). It reads the same sources as the MCP's tools, so a day's rows agree with what the tools answered that day, and anyone can run it. A new day is added each day after 00:30 UTC.

## Which tokens

Every Robinhood Chain token that BitGo supports, every Robinhood Chain token on the lists of LI.FI, CoinGecko, Trust Wallet and MetaMask, and every token with a Uniswap v3 or v4 pool created on Robinhood Chain (chain id 4663). Tokens that never listed anywhere stay in, so every count includes the tokens that went nowhere.

## Files

| File | What it holds |
| --- | --- |
| `tokens.jsonl` | One row per token above, with its contract address on Robinhood Chain. |
| `events.jsonl` | One row per dated public step of a token, such as BitGo adding support for it, its first mint, its first pool on a Uniswap venue or its entry in a wallet's token list. Rows are only ever added: a published row is never changed or removed. |
| `outcomes.jsonl` | One row per outcome: a token tradable as a Robinhood pair, a Robinhood page for it, or a listing on Bitstamp, KuCoin or Gate. |
| `events.csv`, `outcomes.csv` | The same rows as the two JSONL files, for spreadsheets. |
| `base-rates.json` | Counts of what followed each kind of step. See [Counts, not forecasts](#counts-not-forecasts). |
| `feed.json`, `feed.xml` | Each day's new steps as a [JSON Feed 1.1](https://www.jsonfeed.org/version/1.1/) and as RSS 2.0, so an agent can follow the record by polling one file. |
| `manifests/YYYY-MM-DD.json` | One per day: the SHA-256 and row count of every file published that day. |

## When a step was public, and when it was seen

Rows carry two times, and the source of the first:

- `at` is the earliest time the step can be shown to have been public. For a step on Robinhood Chain it is the time of the block that holds it. For BitGo, Trust Wallet and MetaMask it is when the change reached the master branch of their public code, which can be later than the commit's own date.
- `atFrom` says where `at` comes from: `block`, `push` or `observed`.
- `observedAt` is when this record first saw the row.

A source that publishes no date, such as CoinGecko or KuCoin, gets an `at` equal to `observedAt`, and `atFrom` is `observed`.

For a backtest, cut on `at`: keep the rows whose `at` is at or before your cut-off, and your agent sees only what was public by then. The BrainChain MCP's tools cut the same way. Comparing `observedAt` with `at` shows how long a step took to reach this record.

## Rows from the first run

The first run recorded the history before it in one pass, and its rows carry `backfill: true`. Their `at` comes from the source as on any other row; their `observedAt` is the time of that first run.

An outcome that was already in place at the first run, from a source that gives no date, is dated to that run. Read its `at` as "listed by": the listing happened at that time or earlier.

## Manifests

Each day has one manifest, `manifests/YYYY-MM-DD.json`. The values below are examples:

```json
{
  "date": "2026-11-20",
  "readAt": "2026-11-20T00:31:07Z",
  "headBlock": 41234567,
  "files": {
    "tokens.jsonl": { "sha256": "9f2c…77d0", "rows": 1204 },
    "events.jsonl": { "sha256": "41ab…0c3e", "rows": 5310 }
  }
}
```

- `date` is the UTC day the manifest belongs to.
- `readAt` is when the run read its sources, and `headBlock` the last Robinhood Chain block it read.
- `files` names every file published that day with its SHA-256 and its rows: lines in a JSONL file, lines after the header in a CSV, entries in `base-rates.json` and items in a feed.

A manifest does not list itself. Its own hash goes on BrainChain.

## Check a day against BrainChain

Each day's manifest is hashed with SHA-256, and the hash is written to BrainChain once, as a record of kind `dataset`. BrainChain's operator (`0x70C665780E43BA1496a4E691e408F24844dB58eC`) sends it to the Inscriptions contract on BrainChain mainnet (`0x770D6756CA8a1627FFf01B846383b13d6fa0c514`):

```json
{"brainchain":1,"kind":"dataset","date":"2026-11-20","repo":"usebrainchain/brainchain-mcp-data","manifest":"<SHA-256 of manifests/2026-11-20.json>","commit":"<commit>","rows":{"tokens.jsonl":1204,"events.jsonl":5310},"at":"2026-11-20T00:42:10Z"}
```

`commit` is the commit in this repository that published the day, and `rows` repeats each file's count from the manifest. A day is published in one commit and inscribed once, so its record never changes. To check a day you need [Foundry's](https://getfoundry.sh) `cast`, `jq` and `git`.

1. Read the day's record from BrainChain:

   ```bash
   day=2026-11-20
   cast logs --rpc-url https://rpc.usebrainchain.com --from-block 498 \
     --address 0x770D6756CA8a1627FFf01B846383b13d6fa0c514 \
     'Inscribed(address indexed author, string kind, bytes data)' 0x70C665780E43BA1496a4E691e408F24844dB58eC --json |
     jq -r '.[].data' |
     while read -r data; do cast abi-decode --input 'f(string,bytes)' "$data" | sed -n 2p | cast to-ascii; echo; done |
     jq -cR --arg day "$day" 'fromjson? | select(.kind == "dataset" and .date == $day)'
   ```

2. Check out the commit the record names, and hash the manifest. The result must equal the record's `manifest`:

   ```bash
   git checkout <commit>
   shasum -a 256 manifests/$day.json
   ```

3. Check every file against the manifest. Each line must end in `OK`:

   ```bash
   jq -r '.files | to_entries[] | "\(.value.sha256)  \(.key)"' manifests/$day.json | shasum -a 256 -c
   ```

On Linux, use `sha256sum` in place of `shasum -a 256`.

## Counts, not forecasts

`base-rates.json` has an entry for each kind of step: how many tokens took it (`n`), the share of them with each outcome within 7, 30 and 90 days, the median number of days from the step to the outcome, and the tokens behind each number, so every figure leads back to its rows. It counts what happened to the tokens in this record up to the day it was built. It is not a forecast and gives no score.

A step is a lead to look into. It does not say why it happened, and it does not mean a listing follows. Where `n` is small, a share can move a lot from one day to the next.

## License

MIT. See [LICENSE](LICENSE). Questions and corrections: hello@usebrainchain.com.
