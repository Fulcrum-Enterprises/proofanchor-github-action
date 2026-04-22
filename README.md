# ProofAnchor — Timestamp Release

Anchor a SHA-256 hash of your release artifact to the **Polygon blockchain** via [ProofAnchor](https://proofanchor.com). Immutable, verifiable, permanent.

One line in your workflow. Proof that your release existed at a specific point in time.

## Why

- **Supply chain integrity** — prove your release binary hasn't been tampered with
- **Timestamp evidence** — immutable blockchain proof of when your release was published
- **Zero key management** — no wallet, no private keys, no crypto knowledge needed
- **Automatic** — runs on every release, appends verification link to release notes

## Quick Start

```yaml
name: Timestamp Release
on:
  release:
    types: [published]

jobs:
  timestamp:
    runs-on: ubuntu-latest
    permissions:
      contents: write  # needed to update release body
    steps:
      - uses: actions/checkout@v4

      - name: Build release artifact
        run: zip -r dist/release.zip src/

      - name: Timestamp with ProofAnchor
        uses: Fulcrum-Enterprises/proofanchor-github-action@v1
        with:
          api-key: ${{ secrets.PROOFANCHOR_API_KEY }}
          file: dist/release.zip
```

That's it. Your release notes now include a blockchain verification link.

## Inputs

| Input | Required | Description |
|-------|----------|-------------|
| `api-key` | Yes | ProofAnchor API key (starts with `pa_`). Get one at [proofanchor.com/settings](https://proofanchor.com/settings) |
| `file` | One of file/hash | Path to the file to timestamp |
| `hash` | One of file/hash | Pre-computed SHA-256 hash (64 hex chars) |
| `title` | No | Title for the proof (defaults to filename or release tag) |
| `description` | No | Description for the proof |
| `api-url` | No | API base URL (default: `https://proofanchor.com`) |
| `add-to-release` | No | Append proof details to release body (default: `true`) |

## Outputs

| Output | Description |
|--------|-------------|
| `proof-id` | Public ID of the anchored proof |
| `hash` | SHA-256 hash that was anchored |
| `verify-url` | Public verification URL |
| `status` | Proof status (`pending`, `anchored`, `failed`) |

## Examples

### Timestamp a release artifact

```yaml
- uses: Fulcrum-Enterprises/proofanchor-github-action@v1
  with:
    api-key: ${{ secrets.PROOFANCHOR_API_KEY }}
    file: dist/myapp-v1.2.3.tar.gz
```

### Timestamp a pre-computed hash

```yaml
- name: Compute hash
  id: hash
  run: echo "hash=$(sha256sum dist/release.zip | awk '{print $1}')" >> "$GITHUB_OUTPUT"

- uses: Fulcrum-Enterprises/proofanchor-github-action@v1
  with:
    api-key: ${{ secrets.PROOFANCHOR_API_KEY }}
    hash: ${{ steps.hash.outputs.hash }}
    title: "Release ${{ github.ref_name }}"
```

### Timestamp a commit SHA

```yaml
- uses: Fulcrum-Enterprises/proofanchor-github-action@v1
  with:
    api-key: ${{ secrets.PROOFANCHOR_API_KEY }}
    hash: ${{ github.sha }}
    title: "Commit ${{ github.sha }}"
```

### Use outputs in later steps

```yaml
- name: Timestamp
  id: proof
  uses: Fulcrum-Enterprises/proofanchor-github-action@v1
  with:
    api-key: ${{ secrets.PROOFANCHOR_API_KEY }}
    file: dist/release.zip

- name: Print verification URL
  run: echo "Verify at ${{ steps.proof.outputs.verify-url }}"
```

### Skip release body annotation

```yaml
- uses: Fulcrum-Enterprises/proofanchor-github-action@v1
  with:
    api-key: ${{ secrets.PROOFANCHOR_API_KEY }}
    file: dist/release.zip
    add-to-release: "false"
```

## How It Works

1. Action computes SHA-256 of your file (or uses the hash you provide)
2. Calls `POST /api/v1/proofs` on ProofAnchor with the hash
3. ProofAnchor submits the hash to a smart contract on **Polygon mainnet**
4. The blockchain transaction creates an immutable, timestamped record
5. Anyone can verify the proof at the public verification URL — no account needed

## Verification

Every proof can be independently verified:

- **Web:** Visit the verification URL (e.g., `https://proofanchor.com/verify/abc123`)
- **API:** `GET https://proofanchor.com/api/v1/verify/<sha256-hash>` (no auth required)
- **On-chain:** Query the smart contract directly on Polygon (`0xEC24d9322E28e95D1b479C57564bae83af70766e`)

## Getting an API Key

1. Sign up at [proofanchor.com](https://proofanchor.com)
2. Go to Settings > API Keys
3. Create a new key (starts with `pa_`)
4. Add it as a repository secret: Settings > Secrets > `PROOFANCHOR_API_KEY`

Free tier includes 5 proofs. Pro ($9.99/mo) includes 50/month. Business ($49.99/mo) includes 500/month.

## Requirements

- A ProofAnchor account with an API key
- `jq` and `curl` (pre-installed on all GitHub-hosted runners)
- `permissions: contents: write` if using `add-to-release: true`

## License

MIT

## Links

- [ProofAnchor](https://proofanchor.com) — Prove you had it first
- [API Documentation](https://proofanchor.com/api/v1)
- [Fulcrum Enterprises](https://fulcrumenterprises.com) — Building the foundation for an AI driven future
