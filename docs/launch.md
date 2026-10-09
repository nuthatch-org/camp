# engine.camp is live: a free, tip-fresh data engine for Arbitrum One

*Decoded protocol events, raw SQL, live dashboards. No signup, no API key, no quota dashboard. Indexed at chain tip on a laptop.*

---

We've been quietly running a small Arbitrum indexer in the corner of the office for a few months. Today it has a name, a domain, and an open invitation: **[engine.camp](https://engine.camp)**.

camp is a Dune-class data API for Arbitrum One. Same query shape Dune offers — decoded protocol tables, parameterised aggregates, ad-hoc SQL — but live (tip-fresh, not the four-to-six-hour lag Dune ships with) and free at every tier we've thought about offering. It's a single REST surface over a self-hosted Amp node, plus a thin Vercel-hosted gateway that does TLS, rate limits, and a handful of opinionated decoded views.

The whole stack — Next.js gateway, ops scripts, runbook — is open under [nuthatch-org/camp](https://github.com/nuthatch-org/camp) (MIT). Everything in this post is something you can read, run, or fork tonight.

---

## The Dune-shaped hole

Dune is great. We use it. But three things kept biting us, and any team that ships analytics for an Arbitrum-native protocol will have hit the same wall:

1. **Lag.** Dune's pipelines run on a schedule. By the time the decoded `uniswap_v3.swap_events` table catches up to your latest swap, your bot, alert, or dashboard is already wrong.
2. **Coverage.** Dune has decoded tables for protocols the community has built spell-models for. Newer protocols, or anything Arbitrum-specific (Graph Horizon, GMX V2's `EventEmitter` pattern, smaller perp DEXs), aren't covered until someone gets around to it.
3. **Cost at scale.** The free tier is fine for one-off queries. Anything that hits the API regularly — a public dashboard, a live whale-alert bot, an indexer-of-indexers — pushes you up the pricing curve fast.

camp is shaped around those three pain points:

- **Tip-fresh.** Our backfill runs at hundreds of blocks/sec; once it catches up, we're tracking the tip with a few seconds of lag. `/v1/status` shows you exactly how fresh.
- **Decoded by construction.** ampd ships an `evm_decode_log` UDF that takes a topic-bundle and an ABI and gives you typed columns. We don't need to write spell-models — any contract with a public ABI is one query away from decoded.
- **Free, with rate limits sized for real apps.** Anonymous users get 30 req/min and 500 req/hour. Mint a free anonymous token (no email, no signup, takes one `POST`) and you bump to 300/min and 5,000/hour.

---

## What you can do with camp today

Here's the surface, grouped by what you'd actually use it for. All of these are live at `https://engine.camp/v1/...`. The browsable equivalent of every endpoint lives under `/explore/`.

### Decoded protocol events

- **ERC-20 transfers** — `/v1/transfers` with typed `from` / `to` / `value` (Decimal128, not padded hex). Filterable by block range, token contract, address.
- **Generic event logs** — `/v1/events` with the topic-array filter you'd expect, plus optional ABI-driven decoding.
- **Uniswap V3** — `/v1/uniswap-v3/swap`, `/mint`, `/burn` per pool. Decoded ticks, amounts in token decimals, no pad-and-pray.
- **Graph Horizon** — `/v1/horizon/{event}` dispatches 12 decoded events covering provisions, delegations, stake, and slashing. The catalog is at `/v1/horizon`.

### Lookups

The boring stuff that every block explorer offers, but as JSON, no rate limit hassles, no signup:

- `GET /v1/block/{n}` — one block
- `GET /v1/tx/{hash}` — one transaction with receipts
- `GET /v1/address/{a}/tx` — paginated tx list for any address
- `GET /v1/address/{a}/transfers` — every ERC-20 the address has moved or received
- `GET /v1/address/{a}/interactions` — every contract the address has touched, with counts

### Aggregates and tip-fresh views

- `GET /v1/gas/blocks?bucket=1m` — base-fee and throughput per bucket
- `GET /v1/contract/{a}/activity?bucket=1h` — log-count time series for any contract
- `GET /v1/token/{a}/volume?bucket=1d` — transfer volume per bucket for any ERC-20
- `GET /v1/whales/transfers?token=...&min_value=...` — live feed of big Transfers across the major tokens

### Raw SQL

This is the one we love. `POST /v1/sql` takes a single read-only DataFusion `SELECT` against the indexed parquet tables. Constraints to keep your laptop laptop-shaped:

- One statement, `SELECT`-only.
- Must reference `block_num` somewhere (range filter or join). Keeps the scan bounded.
- Hard `LIMIT 1000`, hard 8-second timeout.
- Allowlisted tables, allowlisted UDFs.

In return you get most of the SQL Dune offers — `date_trunc`, `arrow_cast`, `TRY_CAST`, plus ampd's `evm_decode_log`, `evm_topic`, and `eth_call` UDFs. A self-describing contract lives at `GET /v1/sql`.

### Dashboards

We built [/explore](https://engine.camp/explore) as the visual proof of life for every endpoint. Same data, no curl required:

- [/explore/sql](https://engine.camp/explore/sql) — a Dune-style playground (Ctrl+Enter to run)
- [/explore/uniswap-v3](https://engine.camp/explore/uniswap-v3) — decoded swap / mint / burn per pool
- [/explore/horizon](https://engine.camp/explore/horizon) — Graph Horizon timeline with severity accents
- [/explore/whales](https://engine.camp/explore/whales) — big-Transfer ticker, switchable token and threshold
- [/explore/gas](https://engine.camp/explore/gas) — base-fee + throughput
- [/explore/token](https://engine.camp/explore/token) — bucketed volume + recent transfers
- [/explore/address](https://engine.camp/explore/address) — wallet profile (tx, transfers, interactions)
- [/explore/contract](https://engine.camp/explore/contract) — log-count time-series
- [/explore/lookup](https://engine.camp/explore/lookup) — ad-hoc block / tx / events forms
- [/explore/signatures](https://engine.camp/explore/signatures) — well-known topic0 reference

If you want to see what camp can do in five minutes, open `/explore/sql`, hit Run, and start poking.

---

## 30-second quickstart

```bash
# 1. Tip check — how fresh are we?
curl https://engine.camp/v1/status

# 2. Last 100 USDC transfers
curl "https://engine.camp/v1/transfers?token=0xaf88d065e77c8cc2239327c5edb3a432268e5831&limit=100"

# 3. Ad-hoc SQL
curl -X POST https://engine.camp/v1/sql \
  -H "Content-Type: application/json" \
  -d '{
    "sql": "SELECT date_trunc('"'"'hour'"'"', block_timestamp) AS h, COUNT(*) AS swaps FROM uniswap_v3_swaps WHERE block_num > 466000000 GROUP BY 1 ORDER BY 1 DESC LIMIT 24"
  }'
```

Three curls and you've checked freshness, pulled decoded transfers, and run hourly Uniswap V3 swap counts for the last day. Zero signup.

---

## Inside the engine

The architecture is intentionally boring. We picked components that already work and stitched them together.

```
client
  ↓
engine.camp                         (Vercel edge: TLS, DDoS, CDN cache)
  ↓
Cloudflare Quick Tunnel             (private origin link;
  ↓                                  URL auto-rotates and re-syncs to Vercel env)
nginx :1604                         (shared-secret + Redis rate limit)
  ├─ /         → Flight shim :1606  (JSONL ⇆ Arrow Flight bridge)
  ├─ /srh/     → Redis HTTP shim    (rate-limit state)
  └─ /healthz
  ↓
ampd (camp-node — self-built engine, v0.1.0; Arbitrum One indexer; Flight on :1702; parquet on local SSD)
```

Three things deserve explanation.

### ampd

ampd is what does the real work: Rust, designed for SQL-against-EVM. It pulls blocks/receipts/logs from an RPC, writes them to Parquet on disk, exposes everything through Apache Arrow FlightSQL (and a JSON Lines HTTP server), and ships with the EVM-specific UDFs that make decoded queries possible. Compactor merges small parquets into bigger ones in the background; once that catches up, narrow-range queries return in well under a second.

camp builds `ampd` **from source — [`nuthatch-org/camp-node`](https://github.com/nuthatch-org/camp-node)** (built on Edge & Node's Amp, BUSL-1.1) — not the closed-source binary from ampup.sh. We point it at a single dataset, `_/arbitrum_one`, and let it tip-follow Arbitrum One. The usable window grows by ~24h every calendar day; we let history accumulate rather than bulk-backfilling.

### The Flight shim

The fork's `ampd` serves both Arrow Flight and JSON Lines natively, but camp currently routes through a thin Flight ⇆ JSONL bridge (`flight-shim`, in the amping ops repo) — nginx terminates the JSONL side and proxies to the Flight side over localhost. (The shim is now optional: the gateway could point straight at the fork's native JSONL server.)

The shim handles all the awkward type conversions — Arrow Decimal128 with overflow, struct-of-Utf8 from `evm_decode_log`, Binary columns the JSON encoder doesn't know what to do with. The gateway gets back clean JSON; the shim ate the Arrow.

### Caching, rate limiting, and the tunnel

- **Edge caching** — Vercel's CDN handles the bulk of read traffic. Anything that's safe to cache (block lookups, signatures, openapi.yaml) sits at the edge with sensible TTLs.
- **Rate limits** — Upstash-API-compatible HTTP shim over a local Redis container, exposed via the same tunnel under `/srh/`. The gateway uses `@upstash/ratelimit` for sliding windows. Anonymous tier is IP-keyed; the token tier is token-keyed.
- **Cloudflare Quick Tunnel** — gives us a stable `https://` origin without exposing the laptop to the internet. The tunnel URL rotates on restart, so we have a small script that watches for rotations and PATCHes both `AMP_ORIGIN` and `UPSTASH_REDIS_REST_URL` in Vercel, then triggers a redeploy. End-to-end recovery from a tunnel restart is about 30 seconds.

---

## Anonymous tokens for higher limits

We didn't want signups. We also didn't want bots to vacuum the whole API and starve everyone else. The compromise is **anonymous bearer tokens**:

```bash
# Mint a token (no email, no PII; 5 mints/day per IP)
curl -X POST https://engine.camp/v1/tokens

# Use it
curl https://engine.camp/v1/status \
  -H "Authorization: Bearer camp_<your-token>"
```

The mint endpoint is itself IP-rate-limited (5/day), which is the Sybil-cost. Tokens are opaque (`camp_<32 base32 chars>`, ~160 bits of entropy), stored as JSON in Redis with a 30-day sliding TTL — every successful request resets the clock, so active users keep their token; abandoned tokens expire on their own.

| Tier | Per minute | Per hour |
|---|---|---|
| Anonymous (no token) | 30 | 500 |
| With token | 300 | 5,000 |

Invalid token = 401. We don't silently downgrade to anonymous — better that you notice the token is dead than burn through your IP quota without knowing why.

Tokens are inspectable at `GET /v1/tokens/me` (returns prefix, created_at, TTL, tier). They're not retrievable — if you lose one, mint another.

---

## Why we can do this for free

Honest answer: because Arbitrum data is cheap to host once you've solved the indexing problem, and because we benefit from people using it.

The whole stack runs on:

- One ThinkPad in the office (the actual production host)
- A free Cloudflare account (Quick Tunnel + their edge for the origin link)
- Vercel's Hobby tier for the public-facing Next.js app
- One small Postgres instance for ampd's metadata
- ~16 GB of Parquet on the laptop's SSD for ~24h of Arbitrum

Total monthly cost: somewhere around $0. The marginal cost of one more user is negligible. We rate-limit not because requests are expensive but because the laptop is a laptop, and we'd rather not get sucker-punched by a runaway script.

Long-term: if it grows, we'll move ampd to a slightly bigger box and keep the same architecture. If it grows a lot, we'll figure that out then. There's no monetisation plan, no pricing page, no eventual paywall waiting to be unlocked.

---

## Run your own camp

camp is a recipe, not a service. If you want your own — for a different chain, for higher limits, for compliance reasons, for fun — every piece is documented and scripted.

### The five moving parts

1. **An Arbitrum RPC.** Anything that supports `eth_getLogs` and `eth_getBlockByNumber` works. We use Lodestar's own RPC for ours; an Alchemy or Infura key works too. Free tiers are usually fine for the backfill if you're patient.
2. **A server.** Anything Linux-ish with a few hundred GB of disk and enough RAM for ampd (we get away with 16 GB). A ThinkPad is fine. A NUC is fine. A small VPS is fine.
3. **ampd.** Single binary, configured via TOML, exposes FlightSQL on a local port. Deploys as a systemd unit. The amping repo has working configs (`ampd.toml`) and the dataset manifest (`amp.config.ts`) you can copy.
4. **The Flight shim.** Optional today, mandatory if you want to expose ampd over a network that speaks anything other than Arrow Flight. Our shim is in `~/amping/flight-shim/` — it's small enough to read in one sitting.
5. **The gateway.** The Next.js 16 app at [nuthatch-org/camp](https://github.com/nuthatch-org/camp). `pnpm install && pnpm dev` works locally. Set `AMP_ORIGIN` to point at your shim and you're live.

### Wiring it up

The [amping](https://github.com/cargopete/amping) repo (it's private; we'll open-source the relevant bits as we go) ships a working set of deploy scripts under `deploy/`. The runbook in `RUNBOOK.md` covers the unhappy paths: tunnel rotation, ampd crash, compactor overload, rate-limit shim down, Vercel deploy out of sync.

The whole bring-up, end-to-end on a clean box, is roughly:

```bash
# Server-side
./deploy/setup-local.sh           # ampd + Postgres + nginx + Redis + Flight shim
./deploy/setup-tunnel.sh          # Cloudflare named tunnel (or Quick Tunnel)
./deploy/deploy-dataset.sh        # backfill ampd from your chosen start block

# Gateway-side
git clone https://github.com/nuthatch-org/camp
cd camp && pnpm install
# set AMP_ORIGIN + UPSTASH_REDIS_REST_URL in Vercel env
vercel --prod
```

If you point it at a different chain, you'll need to change one constant (`AMP_DATASET`) and tweak the protocol-specific routes that assume Arbitrum-specific contracts. More on that below.

---

## Fork it

camp is designed to be modified. Three common shapes of fork:

### Switch chains

ampd supports any EVM chain. To run camp on, say, Base:

- Deploy a Base dataset to your ampd (`./deploy/deploy-dataset.sh base@1.0.0`).
- Set `AMP_DATASET=_/base@1.0.0` in the gateway env.
- Remove or replace the Arbitrum-specific endpoints (Horizon, the specific Uniswap V3 pool list).

The generic surface — `/v1/transfers`, `/v1/events`, `/v1/sql`, `/v1/gas/blocks`, address/contract lookups — works on any chain unchanged.

### Decode a new protocol

We use a registry pattern. Look at `src/app/v1/horizon/_events.ts` and `src/app/v1/uniswap-v3/_pools.ts` for the shape: a typed event registry, an ABI fragment per event, and a generic decoded-event route that dispatches on the registry.

To add, say, a perp DEX:

1. Pull the ABI for the events you care about.
2. Add an entry to the registry: name, topic0, ABI fragment, decoded column types.
3. Copy `src/app/v1/uniswap-v3/[event]/route.ts` and point it at your registry.

Live in 30 minutes if you know the protocol.

### Add a new aggregate

Aggregates are even cheaper. `src/app/v1/gas/blocks/route.ts` is the template: parse query params, build a SQL string with strict allowlisting, send it to the shim, return JSON.

A "transactions per block-builder per hour" aggregate is twenty lines of new code.

---

## What's next

The roadmap lives in [ROADMAP.md](https://github.com/nuthatch-org/camp/blob/main/ROADMAP.md) and updates with every release. The honest near-term list:

- **GMX V2 decoded events.** GMX uses an `EventEmitter` pattern where every event is encoded into a single generic log, which means `evm_decode_log` needs a registry of expected payloads rather than per-topic ABI fragments. Different shape than the rest of our decoded protocols, but high-value for Arbitrum analysts.
- **CSV / Arrow IPC export formats.** Right now everything returns JSON. CSV for Excel-and-Sheets users; Arrow IPC for pandas/Polars users who want zero-copy.
- **Native Amp CDC bridge.** ampd ships native CDC events (Insert / Delete / Reorg). We have a polling-based `/v1/stream/blocks` SSE today; the real win is plumbing CDC through the Flight shim and exposing it as `/v1/stream/transfers?token=...`, `/v1/stream/whales?...`, and ultimately `/v1/stream/sql?q=...` (a streaming query subscription).
- **Webhooks.** "POST to my URL when a matching event arrives." Built on top of the streaming layer.
- **A second chain.** Once Arbitrum protocol coverage feels real (GMX V2, Camelot V3, a few more lending protocols), we'll add Base or Optimism. The architecture is chain-agnostic; the protocol coverage is where the work is.

Things we're deliberately *not* doing: a paid tier, multi-tenant dashboards, free-tier limits beyond what's needed to protect the laptop, balance endpoints that aren't SQL-shaped. camp is a data API. If you want a hosted dashboard tool, point Grafana / Hex / Metabase at `/v1/sql` and you have one.

---

## Open work, open invite

Everything is open. The Next.js gateway, the ampd configs, the ops scripts, the dashboard UI. Read the code, file issues, send PRs, ask weird questions:

- **Try it:** [engine.camp](https://engine.camp)
- **Docs:** [engine.camp/docs](https://engine.camp/docs) and [openapi.yaml](https://engine.camp/openapi.yaml)
- **Code:** [github.com/nuthatch-org/camp](https://github.com/nuthatch-org/camp)
- **Background:** [intro to camp](https://www.lodestar-dashboard.com/blog/camp-free-amp-api-arbitrum) and [camp deep dive](https://www.lodestar-dashboard.com/blog/camp-deep-dive) over on the Lodestar blog
- **Roadmap:** [ROADMAP.md](https://github.com/nuthatch-org/camp/blob/main/ROADMAP.md)

If you find a query that should be fast and isn't, an endpoint that returns surprising data, or a protocol you'd love to see decoded — open an issue. We read all of them, and the laptop is a phone call away from a fix.

camp is what happens when we admit that Arbitrum data has been an unmet need in our own work and decided to ship the smallest thing that would solve it. We hope it solves yours, too.

Welcome to engine.camp.
