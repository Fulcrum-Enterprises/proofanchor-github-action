# ProofLedger — Timestamp Release

Anchor a SHA-256 hash of your release artifact on the **Polygon** blockchain via [ProofLedger](https://proofledger.io).

One step in your workflow. Independent proof that your release existed at a specific point in time, verifiable by anyone without a ProofLedger account.

> **Renamed from ProofAnchor (August 2026).** ProofAnchor was retired and consolidated into ProofLedger. Versions before `v2` called `proofanchor.com`, which no longer serves an API. The `v1` tag now points at this code, so `@v1` keeps working. A workflow pinned to an older commit SHA still calls proofanchor.com: move it to `@v2`.

## Why

- **Supply chain integrity.** Show that a published binary is byte-for-byte the file your workflow anchored at release time.
- **Timestamp record.** A public, tamper-evident record on Polygon of when the release existed, independent of your repo, your CI logs and your own servers.
- **Your file never leaves the runner.** Only the SHA-256 digest and a label (the file name, or the `filename` input) are sent. ProofLedger never receives the artifact. The label appears on the public certificate, so if the file name is private, set `filename` to a neutral label.
- **Verifiable by anyone.** No account and no API key. Keep the proof's bundle with the release and anyone can check it offline with the [`verify-proof`](https://pypi.org/project/verify-proof/) package on PyPI, and look the Polygon transaction up on any block explorer, with no cooperation from ProofLedger.
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

Your release notes now carry the hash, the proof ID, and ProofLedger's public lookup command, which anyone can run without an account. Once the anchor lands (usually within a minute), the lookup names the Polygon transaction, which anyone can check on a block explorer.

## Getting an API key

API keys are issued from **Account → API Keys** at [proofledger.io](https://proofledger.io) on any plan, including Free, and start with `sk_`. Store it as a repository secret, never in the workflow file.

Each run creates one proof, which counts toward your plan's monthly API allowance (100 on the free plan; see [pricing](https://proofledger.io/pricing.html)). Every proof is anchored on Polygon. Bitcoin anchoring was discontinued on 2026-10-15: the `bitcoin` input is still accepted so existing workflows keep running, and it is ignored.

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `api-key` | yes | | ProofLedger API key (`sk_...`). |
| `file` | one of | | Path to the file to timestamp. Mutually exclusive with `hash`. |
| `hash` | one of | | Pre-computed 64-character SHA-256 hex digest. Mutually exclusive with `file`. |
| `filename` | no | filename or `Release <tag>` | Label stored with the proof. It appears on the public certificate and in your dashboard. |
| `bitcoin` | no | `false` | Deprecated and ignored. Bitcoin anchoring ended on 2026-10-15; every proof is anchored on Polygon. |
| `api-url` | no | `https://proofledger.io` | Override for staging. |
| `add-to-release` | no | `true` | Append proof details to the GitHub release body. |

## Outputs

| Output | Description |
|--------|-------------|
| `proof-id` | ProofLedger proof ID. |
| `hash` | The SHA-256 that was submitted for anchoring. |
| `verify-url` | ProofLedger's public lookup URL for this hash. |
| `status` | Polygon anchoring status (`ANCHORED` or `PENDING`). |
| `evidence-readiness` | Anchoring readiness reported by ProofLedger, e.g. `PENDING_POLYGON` or `READY_POLYGON`. The output keeps this name so existing workflows do not break. |

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

## Verifying a proof

ProofLedger's public lookup shows whether a hash has a proof, with no key and no account:

```bash
curl "https://proofledger.io/api/v1/verify?hash=<sha256>"
```

To check the record itself, look up the Polygon transaction on any block explorer, or verify offline, without contacting ProofLedger at all, against the proof's bundle (fetch it once with `GET /api/v1/proof/<id>/bundle` and your key, and keep it next to the release):

```bash
pip install verify-proof
verify-proof verify ./release.zip --proof bundle.json
```

There is also a verification page at [proofledger.io/verify](https://proofledger.io/verify) where you can paste the hash.

## What this does and does not prove

It proves that a specific SHA-256 digest existed no later than the block it was anchored in. That is a statement about **timing**, and it is the part that is normally hardest to establish after a dispute starts.

It does not prove who created the file, that the file is original, or that its contents are true. Those are separate questions and a timestamp does not answer them.

## Failure modes

The step fails loudly rather than silently on:

- `401` / `403`: the key is missing or malformed.
- `429`: on the Free plan, this month's 100 API proofs are used up (paid plans continue at $0.01 a proof), or the key sent more than 120 requests in a minute.
- Any other non-2xx, with the response body in the log.

## License

MIT. Copyright Fulcrum Enterprises LLC.
