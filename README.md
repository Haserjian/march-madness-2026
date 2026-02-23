# March Madness 2026 -- Locked Predictions

Daily predictions for every NCAA tournament game, digitally signed and publicly published before tipoff. Anyone can verify that no prediction was changed after publishing.

**Not blockchain. Not cryptocurrency.** This uses digital signatures (Ed25519) -- the same technology that secures SSH connections and software updates. Standard, well-tested, boring math.

## Why this exists

Anyone can claim they predicted a game correctly -- after it's over. That's cheap. This project makes predictions **verifiably pre-game**: each day's picks are digitally signed and publicly published before tipoff, and the signature breaks if even one prediction is changed afterward.

Think of it like a notarized timestamp for your picks -- except the notary is math, not a person. The result: no hindsight editing, and a public track record anyone can audit.

## How predictions are made

No AI model. The predictions come from **what the sportsbooks collectively think will happen**: consensus odds from every major US bookmaker, averaged together after removing the house's profit margin (called the "vig"). The market's wisdom is the prediction.

**Live scoreboard:** [haserjian.github.io/march-madness-2026](https://haserjian.github.io/march-madness-2026)

Currently running on regular-season games as a pre-tournament dry run. Tournament predictions begin with bracket week.

## Verify any day (requires Python)

Every daily prediction is digitally signed before tipoff. You can check any day's signature yourself -- no trust required:

```bash
# Install the signature library (only dependency)
pip install pynacl

# Fetch and verify (replace date as needed)
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

Replace `2026-02-24` with any date. If the signature doesn't match, the predictions were tampered with. (This won't happen -- breaking Ed25519 would also break most of the internet.)

**Pin the signer key** to make sure the signing key hasn't been secretly swapped. The fingerprint should always be:
```
sha256:81f878680fdfbc8986c66ca8df84519b7d26e368e72ec5db2480160278d5e0d3
```

## How it works under the hood

```
Bookmaker odds (TheOddsAPI)
    |
    v
Consensus probability (averaged across all US books, vig removed)
    |
    v
Digital fingerprint of predictions (SHA-256 -- changes if anything is modified)
    |
    v
Pre-tipoff lock (timestamped, append-only log)
    |
    v
Digital signature (Ed25519 -- proves predictions existed at lock time)
    |
    v
Public GitHub Pages (this repo)
```

Each day produces:
- **Predictions file** -- every game's pick, win probability, spread, and raw bookmaker lines
- **Lock entry** -- digital fingerprint frozen before tipoff
- **Signed proof** -- bundled receipt verifiable with [assay-ai](https://pypi.org/project/assay-ai/)
- **Daily attestation** -- single signed JSON tying all fingerprints together

## Trust model

| What you can verify | How |
|---------------------|-----|
| Predictions weren't changed after tipoff | Digital fingerprint (SHA-256) + Ed25519 signature lock content integrity |
| Signature can't be forged | Ed25519 is the same standard used for SSH keys and software signing |
| Site content matches the signed proof | `site_html_hash` + `site_json_hash` are included in the signed payload |
| Source code is auditable | `source_commit` in the attestation points to the exact code version |

## What this proves (and doesn't)

**The digital signature proves:**
- Predictions were signed before being publicly published
- No prediction was modified after signing (the signature would break)
- The signing key is consistent across all days (verifiable via fingerprint pinning)

**The signature does NOT prove:**
- Exact wall-clock timing -- the signature proves integrity, not timestamps. Timing is evidenced by the public Git commit history and the append-only lock log
- That predictions are accurate, profitable, or worth betting on -- this is not betting advice
- Anything about prediction method quality -- v1 intentionally uses a simple market-consensus baseline

## Glossary

| Term | Plain English |
|------|---------------|
| **Ed25519** | A digital signature algorithm. Like a wax seal that breaks if the letter is opened. Same tech behind SSH keys. Not related to cryptocurrency. |
| **SHA-256 hash** | A digital fingerprint. A short string computed from the full predictions file. If even one character changes, the fingerprint is completely different. |
| **Vig (vigorish)** | The sportsbook's profit margin baked into their odds. We strip it out to get the market's true probability estimate. |
| **Brier score** | Measures prediction accuracy: 0.0 = perfect, ~0.25 is coin-flip territory for balanced binary outcomes, 1.0 = always wrong. Lower is better. |
| **Attestation** | A signed JSON file containing the predictions fingerprint, signature, and public key -- everything needed to verify independently. |
| **Moneyline** | How sportsbooks express odds. +150 means a $100 bet wins $150; -150 means you bet $150 to win $100. |

## Data files

- [`index.json`](https://haserjian.github.io/march-madness-2026/index.json) -- machine-readable scoreboard
- [`attestations/YYYY-MM-DD.json`](https://haserjian.github.io/march-madness-2026/attestations/2026-02-24.json) -- daily signed proof

## Run your own

See the [operator runbook](RUNBOOK.md) for how to reproduce the daily pipeline.

## Built with

- [assay-ai](https://pypi.org/project/assay-ai/) -- evidence compiler for AI systems
- [TheOddsAPI](https://the-odds-api.com/) -- bookmaker odds aggregator
- Ed25519 signatures via [PyNaCl](https://pynacl.readthedocs.io/)
