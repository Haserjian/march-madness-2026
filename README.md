# March Madness 2026 -- Receipted Predictions

Pre-tipoff predictions for every NCAA tournament game, cryptographically signed and publicly verifiable.

No ML model. The market IS the prediction: consensus implied probability from US bookmaker moneyline odds, normalized to remove vig.

**Live site:** [haserjian.github.io/march-madness-2026](https://haserjian.github.io/march-madness-2026)

## Verify in 60 seconds

Every daily prediction is Ed25519-signed before tipoff. You can verify any day's attestation with zero trust in us:

```bash
# 1. Install PyNaCl (the only dependency)
pip install pynacl

# 2. Fetch and verify
curl -s https://haserjian.github.io/march-madness-2026/attestations/2026-02-24.json | python3 -c "
import base64, json, sys
att = json.load(sys.stdin)
unsigned = {k: v for k, v in att.items()
            if k not in ('signature', 'signer_pubkey', 'signer_pubkey_fingerprint')}
canonical = json.dumps(unsigned, sort_keys=True, separators=(',', ':')).encode()
from nacl.signing import VerifyKey
vk = VerifyKey(base64.b64decode(att['signer_pubkey']))
vk.verify(canonical, base64.b64decode(att['signature']))
print('PASS: signature valid')
print(f'  Date: {att[\"date\"]}')
print(f'  Games: {att[\"n_games\"]}')
print(f'  Predictions hash: {att[\"predictions_hash\"]}')
print(f'  Signer fingerprint: {att.get(\"signer_pubkey_fingerprint\", \"n/a\")}')
"
```

Replace `2026-02-24` with any date to verify that day's predictions.

**Pin the signer key** to detect key substitution -- the fingerprint should always be:
```
sha256:81f878680fdfbc8986c66ca8df84519b7d26e368e72ec5db2480160278d5e0d3
```

## How it works

```
Bookmaker odds (TheOddsAPI)
    |
    v
Consensus probability (vig-removed average across all US books)
    |
    v
SHA-256 hash of predictions manifest
    |
    v
Pre-tipoff lock (append-only, timestamped)
    |
    v
Ed25519-signed proof pack (assay-ai)
    |
    v
Signed daily attestation (predictions_hash + site hashes + source commit)
    |
    v
Public GitHub Pages (this repo)
```

Each day produces:
- **Prediction manifest** -- every game's pick, probability, spread, bookmaker lines
- **Lock entry** -- SHA-256 hash frozen before tipoff
- **Proof pack** -- signed receipt bundle (verifiable with `assay verify-pack`)
- **Attestation** -- single signed JSON binding all hashes together

## Trust model

| Property | Mechanism |
|----------|-----------|
| Predictions can't be changed after tipoff | SHA-256 lock + Ed25519 attestation timestamp |
| Attestation can't be forged | Ed25519 signature, key pinnable by fingerprint |
| Site content matches attestation | `site_html_hash` + `site_json_hash` in signed payload |
| Source code is auditable | `source_commit` SHA in attestation points to exact code |

**Signer key fingerprint (pin this):**
```
sha256:81f878680fdfbc8986c66ca8df84519b7d26e368e72ec5db2480160278d5e0d3
```

## Data files

- [`index.json`](https://haserjian.github.io/march-madness-2026/index.json) -- machine-readable scoreboard
- [`attestations/YYYY-MM-DD.json`](https://haserjian.github.io/march-madness-2026/attestations/2026-02-24.json) -- signed daily attestation

## Run your own

See the [operator runbook](RUNBOOK.md) for how to reproduce the daily pipeline.

## Built with

- [assay-ai](https://pypi.org/project/assay-ai/) -- evidence compiler for AI systems
- [TheOddsAPI](https://the-odds-api.com/) -- bookmaker odds aggregator
- Ed25519 signatures via [PyNaCl](https://pynacl.readthedocs.io/)
