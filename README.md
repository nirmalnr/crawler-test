# crawler-test

Signed [DeDi protocol](https://github.com/nfh-trust-labs/DeDi/blob/main/docs/publishing-dedi-files.md)
fixtures, hosted on GitHub Pages, for testing
[`dedi-crawler`](https://github.com/nfh-trust-labs/dedi-crawler) end to end against real HTTPS
requests instead of local files.

**None of this is real.** Every domain, publisher name, and key is a throwaway fixture. Private
signing keys live only in a local, gitignored `keys/` directory — never in this repo.

## How to crawl this repo

`domains.txt` lists each fixture as a GitHub Pages *path* (`nirmalnr.github.io/crawler-test/<name>`),
not a bare hostname — the crawler treats each path as if it were its own domain and looks for
`https://<entry>/.well-known/dedi.index.json`. Because these entries aren't hostname-shaped, crawling
this list requires:

```yaml
discovery:
  seeds:
    - https://nirmalnr.github.io/crawler-test/domains.txt
  allow_non_standard_domains: true
```

`allow_non_standard_domains: true` also disables the domain-binding check in `internal/verify`
(`manifest.domain`/`publisher.domain`/`namespace` no longer have to equal the crawled path). That's
why every manifest and file below carries its own dummy-but-realistic-looking domain (e.g.
`beckn-mobility-network.example`) that's completely different from the GitHub Pages path it's
actually served from. **Never set this flag outside of testing against fixtures like these.**

(This repo also has a `.nojekyll` file at the root — without it, GitHub Pages' default Jekyll build
silently drops dotfiles/dot-directories, and every `.well-known/dedi.index.json` 404s.)

## Good examples

Four publishers, together covering all 7 registry schemas from
[decentralized-directory-protocol/schemas](https://github.com/LF-Decentralized-Trust-labs/decentralized-directory-protocol/tree/main/schemas).
Everything here verifies as `Verified`.

| Domain (path in `domains.txt`) | Manifest `domain` | Registries | What's in it |
|---|---|---|---|
| `beckn-mobility-network.example` | `beckn-mobility-network.example` | `beckn_subscriber`, `beckn_subscriber_reference` | 2 subscriber records (a BAP and a BPP) + 2 reference records pointing back at them |
| `civic-trust-registry.example` | `civic-trust-registry.example` | `membership`, `revoke` | 2 membership records + 2 revoked-membership records |
| `keychain-authority.example` | `keychain-authority.example` | `public_key` | 2 entity public-key records, one with a `previousKeys` rotation history |
| `open-data-commons.example` | `open-data-commons.example` | `public-data-set`, `public-rule-set` | 2 datasets (one `data_inline`, one `data_url`+checksum) + 2 rulesets (one `data_inline` rego, one `data_url`+checksum jsonlogic) |

## Mixed example

One manifest, two registries — only one of them should actually get ingested.

| Domain | Registry | Expected outcome |
|---|---|---|
| `mixed-results-registry.example` | `current-members` | `Verified` — ingested |
| `mixed-results-registry.example` | `stale-blacklist` | `Rejected` / `file_expired` — signed correctly, but `next_update` lapsed; **skipped** |

The manifest itself is fine — this demonstrates that verification is per-file, not all-or-nothing
per domain.

## Bad examples

Everything below is intentionally broken. Each is built by taking a validly-signed document and
corrupting exactly one thing, so the failure is isolated to a single check in
`internal/verify`. All of these have been run through `verify.Manifest`/`verify.File` directly (both
from local files and live over HTTPS) to confirm they fail with the exact reason listed.

### Broken manifests

These fail before any of the domain's files are ever examined.

| Domain | What's wrong | Expected outcome |
|---|---|---|
| `expired-authority.example` | `next_update` is in the past | `Rejected` / `manifest_expired` |
| `malformed-manifest.example` | The file isn't valid JSON (truncated) | `Rejected` / `unparseable` |
| `tampered-manifest.example` | Signed, then a field was edited afterward | `Rejected` / `signature_invalid` |
| `unknown-signer.example` | Signed, then the signing key was stripped from `keys[]` | `Rejected` / `verification_method_mismatch` |
| `rsa-key-authority.example` | Signed, then the key's `kty` was changed from `OKP`/Ed25519 to `RSA` | `Rejected` / `publisher_key_invalid` |

### Broken files (`bad-registry-examples.example`)

One valid, verifiable manifest — every file it references is broken a different way.

| File | What's wrong | Expected outcome |
|---|---|---|
| `dedi.tampered.json` | Signed, then a record's `details` was edited afterward | `Rejected` / `signature_invalid` |
| `dedi.expired.json` | `next_update` is in the past | `Rejected` / `file_expired` |
| `dedi.inactive.json` | `registry.state` is `inactive` | `Rejected` / `registry_inactive` |
| `dedi.rotated-key.json` | Validly signed, but by a key never listed in this manifest's `keys[]` | `IntegrityOnly` / `key_not_in_manifest` |
| `dedi.schema-violation.json` | A record is missing a field the registry's own schema requires | `Rejected` / `record_schema_violation` |
| `dedi.bad-key-type.json` | Signed, then `publisher.key.kty` was changed to `RSA` | `Rejected` / `publisher_key_invalid` |
| `dedi.verification-method-mismatch.json` | Signed, then `proof.verification_method` was changed to a key ID that doesn't match `publisher.key.kid` | `Rejected` / `verification_method_mismatch` |
| `dedi.schema-url-unresolved.json` | `registry.schema` is a URL reference instead of inline — the crawler never fetches external schema URLs | `Rejected` / `schema_not_resolved` |
| `dedi.invalid-schema.json` | The inline JSON Schema itself is malformed (`required` is a string, not an array) | `Rejected` / `schema_invalid` |
| `dedi.malformed.json` | The file isn't valid JSON (truncated) | `Rejected` / `unparseable` |

### Not covered here

Two `internal/verify` reasons aren't demonstrated by any fixture in this repo:

- **`domain_mismatch` / `manifest_domain_mismatch`** — require running *without*
  `allow_non_standard_domains`, which every fixture here depends on to be crawlable at all as a
  GitHub Pages path.
- **`canonicalization_failed`** — not practically constructible by hand-editing otherwise-valid JSON.
