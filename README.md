# simplewall-packs

Signed rule packs for the simplewall composite fork
(github.com/PhilopaterSh/simplewall). Consumed by the app's composite
packs module (FR-08/FR-09, SEC-01, DATA-02).

## Pack format (DATA-02, canonical)

```
packs/<pack-id>/pack.json   # UTF-8, canonical machine-generated JSON
packs/<pack-id>/pack.sig    # detached ECDSA P-256 signature (raw 64-byte r||s)
                            # over the EXACT pack.json bytes
```

### pack.json schema (strict — the app refuses anything else)

```json
{
  "id": "telemetry-refresh",
  "version": "1.0.0",
  "key_id": "qw-packs-2026-11",
  "rules": [
    { "path": "c:\\path\\app.exe", "direction": "out|in|both" },
    { "port": 443, "protocol": "tcp|udp", "direction": "out" },
    { "address": "1.2.3.4", "direction": "both" }
  ]
}
```

- Every rule object carries EXACTLY one of `path` / (`port`+`protocol`) /
  `address`, plus `direction`.
- All pack rules are BLOCK rules at a weight BELOW user rules — explicit
  user consent always outranks a pack.
- Max 0x2000 rules per pack.

## Signing (key-epoch `qw-packs-2026-11`)

- ECDSA P-256 (NIST), 64-byte raw `r||s` signature of the pack.json bytes.
- The app embeds the matching public key (uncompressed x||y). THE APP
  CURRENTLY SHIPS A PLACEHOLDER ALL-ZERO KEY: every pack refuses until
  the owner embeds the release key (safe default — see the app's
  `src/composite_packs.c` `packs_public_key`).
- Signing tooling: any ECDSA P-256 signer (OpenSSL:
  `openssl dgst -sha256 -sign key.pem` then convert DER→raw r||s, or
  BCryptSignPayload equivalents). CI signs on push once keys exist.

## Anti-rollback (DATA-02)

The app keeps a per-pack version high-water mark in the non-rebuildable
`pack_watermarks.json` sidecar. Correctly-signed packs with an OLDER
version refuse; the store itself is never rebuilt — a corrupt store makes
packs refuse until the owner re-confirms.

## Packs (FR-09 starter set — content pending keys)

| Pack | Purpose | Status |
|---|---|---|
| Telemetry-Refresh | allow-list for OS update/refresh endpoints | pending signing keys |
| Vendor-blockers | curated known-telemetry vendor blocklist | pending signing keys |
| AI-Agents | controls for local AI-agent egress patterns | pending signing keys |

## Key-rotation runbook (RSK-07)

1. Generate a fresh P-256 keypair; new `key_id` = `qw-packs-<epoch>`.
2. Embed the new public key in the app (release N), keep the old key
   verifying for one release (dual-epoch window).
3. Re-sign all packs under the new epoch; publish.
4. Release N+1: drop the old embedded key.
5. COMPROMISED key: bump epoch immediately, re-sign, and note the
   incident in the app release notes.

## CI

On push: schema-validate every pack.json, verify each detached signature
against the epoch public key, fail the build on any refusal. (Workflow
lands with the first signing key.)
