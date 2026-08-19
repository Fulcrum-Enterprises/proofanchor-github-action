# ProofLedger — Timestamp Release

Anchor a SHA-256 hash of your release artifact to the **Polygon** and, optionally, **Bitcoin** blockchains via [ProofLedger](https://proofledger.io).

One step in your workflow. Independent proof that your release existed at a specific point in time, verifiable by anyone without a ProofLedger account.

> **Renamed from ProofAnchor (August 2026).** ProofAnchor was retired and consolidated into ProofLedger. Versions before `v2` called `proofanchor.com`, which no longer serves an API, so **`v1` no longer works and has been repointed at this code**. If you pinned a commit SHA, move to `@v2`.

## Why

- **Supply chain integrity.** Prove a published binary is byte-for-byte the one you built.
- **Timestamp evidence.** An immutable record of when the release existed, independent of your repo, your CI logs, and your own servers.
- **Your file never leaves the runner.** Only the SHA-256 digest is sent. ProofLedger never receives the artifact.
- **Verifiable by anyone.** No account, no API key, no cooperation from ProofLedger required. Offline verification via the [`verify-proof`](https://pypi.org/project/verify-proof/) package on PyPI.
- **Zero key management.** No wallet, no private keys, no crypto knowledge.

## Quick start

```yaml
name: Timestamp Release
on:
  release:
    types: [published]

jobs:
  timestamp:
    runs-on: ubuntu-latest
    permissions:
      contents: write  # needed to update the release body
    steps:
      - uses: actions/checkout@v4

      - name: Build release artifact
        run: zip -r dist/release.zip src/

      - name: Timestamp with ProofLedger
        uses: Fulcrum-Enterprises/proofanchor-github-action@v2
        with:
          api-key: ${{ secrets.PROOFLEDGER_API_KEY }}
          file: dist/release.zip
```

Your release notes now carry the hash, the proof ID, and a verification command anyone can run.

## Getting an API key

API keys are issued from **Account → API Keys** at [proofledger.io](https://proofledger.io) on the BUSINESS plan, and start with `sk_`. Store it as a repository secret, never in the workflow file.

Polygon anchoring is unlimited on every ProofLedger plan. Bitcoin anchoring is a separate per-anchor escalation and is off unless you ask for it with `bitcoin: true`.

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `api-key` | yes | | ProofLedger API key (`sk_...`). |
| `file` | one of | | Path to the file to timestamp. Mutually exclusive with `hash`. |
| `hash` | one of | | Pre-computed 64-character SHA-256 hex digest. Mutually exclusive with `file`. |
| `filename` | no | filename or `Release <tag>` | Label shown in your ProofLedger dashboard. |
| `bitcoin` | no | `false` | Also queue the proof for Bitcoin anchoring (Merkle-batched daily). |
| `api-url` | no | `https://proofledger.io` | Override for staging. |
| `add-to-release` | no | `true` | Append proof details to the GitHub release body. |

## Outputs

| Output | Description |
|--------|-------------|
| `proof-id` | ProofLedger proof ID. |
| `hash` | The SHA-256 that was anchored. |
| `verify-url` | Public verification API URL for this hash. |
| `status` | Polygon anchoring status (`ANCHORED` or `PENDING`). |
| `evidence-readiness` | ProofLedger's readiness value, e.g. `READY_POLYGON`, `READY_BITCOIN`. |

## Anchoring an existing hash

If you already computed the digest, skip the file:

```yaml
      - name: Timestamp a known digest
        uses: Fulcrum-Enterprises/proofanchor-github-action@v2
        with:
          api-key: ${{ secrets.PROOFLEDGER_API_KEY }}
          hash: ${{ steps.build.outputs.sha256 }}
          filename: my-app-${{ github.ref_name }}.tar.gz
```

## Also anchoring to Bitcoin

```yaml
      - name: Timestamp with Bitcoin escalation
        uses: Fulcrum-Enterprises/proofanchor-github-action@v2
        with:
          api-key: ${{ secrets.PROOFLEDGER_API_KEY }}
          file: dist/release.zip
          bitcoin: true
```

Polygon confirms in seconds. Bitcoin is batched daily via a Merkle tree, so the Bitcoin status stays `PENDING` until the batch is anchored.

## Verifying a proof

Anyone can check a hash against the chain, with no key and no account:

```bash
curl "https://proofledger.io/api/v1/verify?hash=<sha256>"
```

Or verify offline, without contacting ProofLedger at all:

```bash
pip install verify-proof
verify-proof hash ./release.zip
```

There is also a verification page at [proofledger.io/verify](https://proofledger.io/verify) where you can paste a hash or a proof ID.

## What this does and does not prove

It proves that a specific SHA-256 digest existed no later than the block it was anchored in. That is a statement about **timing**, and it is the part that is normally hardest to establish after a dispute starts.

It does not prove who created the file, that the file is original, or that its contents are true. Those are separate questions and a timestamp does not answer them.

## Failure modes

The step fails loudly rather than silently on:

- `401` / `403` — the key is missing, malformed, or not on a plan with API access.
- `429` — the monthly proof limit for your plan has been reached.
- Any other non-2xx, with the response body in the log.

## License

MIT. Copyright Fulcrum Enterprises LLC.
