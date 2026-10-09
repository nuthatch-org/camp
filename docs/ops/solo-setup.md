# Running ampd solo (single-box production)

This is the exact setup `camp` runs in production: **one `ampd` binary, one box,
a local Postgres catalog** — no distributed controller/worker split. It's enough
to index a full chain at tip and serve queries with single-digit-second freshness.

camp builds `ampd` from its own engine, **[nuthatch-org/camp-node](https://github.com/nuthatch-org/camp-node)**
(built on Edge & Node's Amp, BUSL-1.1) — not the closed-source ampup.sh binary. Build it first
(see camp-node's README: `cargo build --release -p ampd -p ampctl`), then follow this guide. Tested
on **camp-node v0.1.0** (`a1937bf`). Note the compactor defaults **off** in
this build — you must enable it explicitly (see [Compaction](#compaction)).

---

## Layout

```
/usr/local/bin/ampd                 # the binary
/etc/ampd/ampd.toml                 # config (below)
/var/lib/ampd/data                  # parquet store
/var/lib/ampd/providers             # provider manifests
/var/lib/ampd/manifests             # dataset manifests
postgres://…/ampd                   # metadata catalog (physical_tables, file_metadata)
```

The Postgres catalog is the **source of truth** for the store. ampd builds scan
plans from the footer stats cached in `file_metadata`; it does **not** re-list
the data directory. Never move, merge, or delete parquet files underneath a live
catalog — see [Don't DIY-compact](#dont-diy-compact-underneath-the-catalog).

---

## `/etc/ampd/ampd.toml`

```toml
# ampd server configuration
data_dir      = "/var/lib/ampd/data"
providers_dir = "/var/lib/ampd/providers"
manifests_dir = "/var/lib/ampd/manifests"

# Bind Flight (gRPC) to localhost only — reverse-proxy external traffic.
# Never expose this port directly to the internet.
flight_addr        = "127.0.0.1:1602"
poll_interval_secs = 3.0

# Optional JSON Lines HTTP endpoint (localhost-only)
# jsonl_addr = "127.0.0.1:1603"

# Admin API — keep on localhost
# admin_api_addr = "127.0.0.1:1610"

[metadata_db]
# Local Postgres is plenty for a single node:
#   sudo -u postgres createdb ampd
#   sudo -u postgres psql -c "CREATE USER ampd WITH PASSWORD '…';"
#   sudo -u postgres psql -c "GRANT ALL ON DATABASE ampd TO ampd;"
url = "postgres://ampd:${AMPD_PG_PASSWORD}@localhost/ampd"

[writer]
compression = "zstd(1)"

# The compactor + collector default to OFF in this build — enable both, or the
# parquet file count grows without bound and cold queries get slow. Other knobs
# (concurrency, intervals, eager limits) fall back to sane build defaults.
[writer.compactor]
active = true

[writer.collector]
active = true
```

`poll_interval_secs = 3.0` is the materialize loop cadence — ingestion *and*
compaction piggyback on it, so on a healthy node you'll see compaction log lines
every few seconds. The `[writer.compactor]`/`[writer.collector]` blocks above are
**required** in this build (unlike the closed v0.0.36 binary, which defaulted the
compactor on in solo mode).

---

## `/etc/systemd/system/ampd.service`

```ini
[Unit]
Description=Amp blockchain database daemon
After=network.target postgresql.service

[Service]
User=ampd
Group=ampd
Environment=AMP_CONFIG=/etc/ampd/ampd.toml
Environment=AMP_DIR=/var/lib/ampd
Environment=HOME=/var/lib/ampd
# camp-node has no `solo` subcommand — `dev` is the all-in-one (Flight + JSON Lines
# + Admin API + an in-process worker), the single-box equivalent.
ExecStart=/usr/local/bin/ampd --config /etc/ampd/ampd.toml dev
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

`LimitNOFILE=65536` matters more than it looks. A node that hasn't compacted yet
holds a **lot** of file descriptors open during cold scans across many small
parquet segments — the default 1024 soft limit will bite you well before the
compactor catches up. Keep the bump even after your file count settles.

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now ampd
journalctl -u ampd -f
```

---

## Compaction

ampd writes many small parquet segments as it ingests, then merges them in the
background. If compaction isn't running, query latency is dominated by
**file-open overhead**, not scan time — e.g. a `logs` query going cold at ~5s
across tens of thousands of ~2 MB files.

**It runs in single-box `dev` mode** once enabled. You do not need a distributed
controller + `ampd worker` split to get compaction — `ampd dev` runs an in-process
worker. But you **must** turn the compactor on in config (`[writer.compactor] active = true`
+ `[writer.collector] active = true`); it is **off by default** in this build.

Confirm it's alive:

```bash
journalctl -u ampd --since "1 hour ago" | grep -i compact
```

Healthy output looks like:

```
compact{table="…logs"}: Scanning 297 segments for compaction
compact{table="…logs"}: Created compaction group (files: 2, ranges: {…})
compact{table="…logs"}: Compaction Success: …/logs/…/<range>.parquet
compaction group completed successfully table=logs
```

(log target: `amp_job_core::materialize::compaction::compactor`)

**If that grep returns nothing**, the compactor isn't enabled. In this build it's
**off by default** — make sure your `ampd.toml` has `[writer.compactor] active = true`
(and `[writer.collector] active = true`) and restart. That config is **required**
here; the file count climbing without bound + no `Compaction Success` lines is the
symptom of leaving it off.

For reference, a healthy store shape (camp, Arbitrum One, ~7.8-day rolling
window, 2.7M blocks): ~4,300 parquet files, median ~1.9 MB, merged groups up to
~50 MB. If your file count climbs without bound and you see no `Compaction
Success` lines, that's the symptom — check your version first.

### Don't DIY-compact underneath the catalog

It's tempting to merge files yourself with duckdb and rewrite paths. Don't. The
Postgres catalog (`file_metadata` with cached footers + `physical_tables`) must
stay consistent with what's on disk. To merge a file safely you'd have to
atomically rewrite `file_metadata` (new paths, row counts, per-column min/max
stats, footer offsets) **and** the `physical_tables` mapping in one transaction —
or queries return missing-path errors and stale results. The native compactor
does exactly that rewrite-plus-catalog-update transactionally; let it.

If you want duckdb for analytics, point it at the compacted parquet **read-only**
and leave the store to ampd. Use duckdb as a read path, never as a mutation path.

---

## Quick health check

```bash
# version + process
ampd --version
ps -o pid,etime,%cpu,%mem,rss -C ampd

# tip freshness (replace with your proxy URL / port)
curl -s http://localhost:1604/v1/status | jq '{latest_indexed_block, latest_indexed_at, blocks_indexed}'

# store size + file count
du -sh /var/lib/ampd/data
find /var/lib/ampd/data -name '*.parquet' | wc -l
```
