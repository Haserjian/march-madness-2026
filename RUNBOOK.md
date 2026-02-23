# Operator Runbook

Daily pipeline for March Madness receipted predictions. Runs from the `ccio` repo (private), mirrors results here (public).

## Prerequisites

```bash
# Clone the source repo (private -- you need access)
cd ~/ccio
source .venv/bin/activate

# API key (free tier: 500 requests/month)
export THEODDS_API_KEY="your_key"

# Verify dependencies
pip install assay-ai pynacl requests
```

## Daily flow

### Before tipoff: predict + lock + attest

```bash
python scripts/march_madness/run_day.py --mirror
```

This runs the full pipeline:
1. Fetches odds from TheOddsAPI
2. Generates predictions (consensus implied probability)
3. Locks the predictions hash (append-only `locks.jsonl`)
4. Attempts settlement (picks up any completed games)
5. Builds signed proof pack
6. Rebuilds static site
7. Signs daily attestation (Ed25519)
8. Mirrors to public repo (`--mirror`)

**Idempotency**: Running twice for the same date skips the prediction step (manifest already exists) and re-runs settle/publish (picks up new scores).

### After games finish: re-settle

```bash
python scripts/march_madness/run_day.py --date YYYY-MM-DD --settle-only --mirror
```

Re-runs settlement to pick up final scores, rebuilds proof pack with settlement receipts, and mirrors updated results.

### Verify any day's attestation

```bash
# Install PyNaCl (the only dependency)
pip install pynacl

# Verify a specific day (replace date as needed)
curl -s https://haserjian.github.io/march-madness-2026/attestations/2026-02-24.json | python3 -c "
import base64, json, sys
att = json.load(sys.stdin)
unsigned = {k: v for k, v in att.items()
            if k not in ('signature', 'signer_pubkey', 'signer_pubkey_fingerprint')}
canonical = json.dumps(unsigned, sort_keys=True, separators=(',', ':')).encode()
from nacl.signing import VerifyKey
vk = VerifyKey(base64.b64decode(att['signer_pubkey']))
vk.verify(canonical, base64.b64decode(att['signature']))
print('PASS:', att['date'], att['n_games'], 'games')
print('Hash:', att['predictions_hash'])
print('Fingerprint:', att.get('signer_pubkey_fingerprint', 'n/a'))
"

# Pin the signer key fingerprint (should always be):
# sha256:81f878680fdfbc8986c66ca8df84519b7d26e368e72ec5db2480160278d5e0d3
```

## Cron setup

For automated daily runs:

```bash
# Edit crontab
crontab -e

# Run at 10:00 AM UTC daily (before most tipoffs)
0 10 * * * /path/to/ccio/scripts/march_madness/cron_run.sh >> /path/to/ccio/data/march_madness/logs/cron.log 2>&1
```

## Pipeline scripts

| Script | Purpose |
|--------|---------|
| `run_day.py` | Full orchestrator (predict -> settle -> publish -> attest -> mirror) |
| `predict_day.py` | Fetch odds, generate predictions, write manifest |
| `settle_day.py` | Fetch scores, match to predictions, compute metrics |
| `publish_day.py` | Convert manifests to receipts, build signed proof packs |
| `build_site.py` | Generate static HTML + JSON for GitHub Pages |
| `attest_day.py` | Sign daily attestation (Ed25519) |
| `mirror_site.py` | Copy site artifacts to public repo and push |
| `verify_day.py` | Standalone verification (Ed25519 signature + hash checks) |

## API credit budget

Free tier: 500 requests/month. Each daily run uses 2 calls (odds + scores). Tournament is ~20 game days = ~40 calls. Pre-tournament daily tracking uses more. Pipeline aborts if credits drop below 50 (configurable with `--min-credits`).

## Troubleshooting

**"THEODDS_API_KEY not set"** -- Export the key in your shell or add to `.zshrc`.

**"Refusing to overwrite existing manifest"** -- Predictions are immutable per day. Use `--force` only for debug/backfill.

**"API credits critically low"** -- Pipeline auto-aborts below threshold. Wait for monthly reset or upgrade plan.

**Settlement shows all PENDING** -- Games haven't finished yet. Re-run with `--settle-only` after games complete.

**Mirror "Push failed"** -- Check GitHub auth (`gh auth status`). The public repo needs push access.
