# crawler-test

Signed [DeDi protocol](https://github.com/nfh-trust-labs/DeDi/blob/main/docs/publishing-dedi-files.md)
fixtures, hosted on GitHub Pages, for testing DeDi Crawlers.

**None of this is real.** Every domain, publisher name, and key is a throwaway fixture. Private
signing keys live only in a local, gitignored `keys/` directory — never in this repo.

## How to crawl this repo

`domains.txt` lists each fixture as a GitHub Pages *path* (`nirmalnr.github.io/crawler-test/<name>`),
not a bare hostname — the crawler treats each path as if it were its own domain and looks for
`https://<entry>/.well-known/dedi.index.json`. Because these entries aren't hostname-shaped, crawling
this list requires setting this as a discovery list.

This works by disabling the domain-binding check. That's
why every manifest and file below carries its own dummy-but-realistic-looking domain (e.g.
`beckn-mobility-network.example`) that's completely different from the GitHub Pages path it's
actually served from. **Never set this flag outside of testing against fixtures like these.**

(This repo also has a `.nojekyll` file at the root — without it, GitHub Pages' default Jekyll build
silently drops dotfiles/dot-directories, and every `.well-known/dedi.index.json` 404s.)

## Good examples

Four publishers, together covering all 7 registry schemas from
[decentralized-directory-protocol/schemas](https://github.com/LF-Decentralized-Trust-labs/decentralized-directory-protocol/tree/main/schemas).
Each file's `registry.schema` is the real, canonical URL to that schema in the protocol repo —
matching the URL-reference style shown in
[the spec's own DeDi file example](https://github.com/nfh-trust-labs/DeDi/blob/main/docs/publishing-dedi-files.md)
— rather than an inline copy.

| Domain (path in `domains.txt`) | Manifest `domain` | Registries | What's in it |
|---|---|---|---|
| `beckn-mobility-network.example` | `beckn-mobility-network.example` | `beckn_subscriber`, `beckn_subscriber_reference` | 3 subscriber records (a BAP and a BPP whose `subscriber_id` is a subdomain of the manifest's domain, plus one rogue record that isn't — see below) + 2 reference records pointing back at the two conformant ones |
| `civic-trust-registry.example` | `civic-trust-registry.example` | `membership`, `revoke` | 2 membership records + 2 revoked-membership records |
| `keychain-authority.example` | `keychain-authority.example` | `public_key` | 2 entity public-key records, one with a `previousKeys` rotation history |
| `open-data-commons.example` | `open-data-commons.example` | `public-data-set`, `public-rule-set` | 2 datasets (one `data_inline`, one `data_url`+checksum) + 2 rulesets (one `data_inline` rego, one `data_url`+checksum jsonlogic) |

Every manifest and file above verifies and gets ingested — confirmed live, by querying
`dedi-crawler`'s DeDi.global lookup API after a real crawl. `internal/verify.File` itself never
fetches a `registry.schema` URL (it only compiles inline schemas), but the caller,
`internal/reconcile.SyncDomain`, always resolves a URL schema first — via `schemaCache.Content` — and
hands `verify.File` the already-fetched, already-compiled result. Whether that URL also happens to
match one of DeDi.global's own canonical schema tags (`mapper.ResolveSchemaTag`) only changes what
gets sent onward to DeDi.global (a tag vs. the raw schema content) — it doesn't affect whether local
verification succeeds. See [Custom schema resolution](#custom-schema-resolution) below for the case
where the URL *doesn't* dereference cleanly.

### `beckn_subscriber`: subscriber_id vs. domain ownership

Real beckn network operators expect a `beckn_subscriber` record's `subscriber_id` to be the
publisher's own domain or a subdomain of it — proving the registrant actually owns the identity
it's claiming, not just any string that satisfies the schema. `dedi.beckn_subscriber.json` reflects
that convention:

| `record_name` | `subscriber_id` | Conforms to `beckn-mobility-network.example`? |
|---|---|---|
| `dummy-transit-bap` | `bap.beckn-mobility-network.example` | Yes — subdomain |
| `dummy-rideshare-bpp` | `bpp.beckn-mobility-network.example` | Yes — subdomain |
| `rogue-imposter-cds` | `mobility.rogue-imposter.example` | **No** — unrelated domain |

**`dedi-crawler`'s own `internal/verify.File` doesn't enforce this** — it only validates records
against `registry.schema`, which (correctly, per the upstream schema) just requires `subscriber_id`
to be *a* string; `verify` has no way to know what domain published the file, and has no concept of
per-record rejection within an otherwise-valid file. But querying the live lookup API after a crawl
confirms `rogue-imposter-cds` is filtered out downstream anyway: `beckn_subscriber`'s
`record_count` comes back `2`, not `3`. Whatever's enforcing this — evidently something in
DeDi.global itself, past the point where `dedi-crawler` submits the record — it's real, just not
visible from reading `dedi-crawler`'s own source.

## Mixed example

One manifest, two registries — only one of them should actually get ingested.

| Domain | Registry | Expected outcome |
|---|---|---|
| `mixed-results-registry.example` | `current-members` | `Verified` — ingested |
| `mixed-results-registry.example` | `stale-blacklist` | `Rejected` / `file_expired` — signed correctly, but `next_update` lapsed; **skipped** |

The manifest itself is fine — this demonstrates that verification is per-file, not all-or-nothing
per domain.

## Custom schema resolution

Another one-succeeds-one-doesn't manifest, this time isolating `internal/reconcile`'s schema
dereferencing specifically (see [Good examples](#good-examples) above). `schemas/`
in this repo hosts a small publisher-controlled schema, `custom-widget-registry.schema.json` —
genuinely fetchable, but not one of DeDi.global's recognized canonical tags.

| Domain | Registry | `registry.schema` | Expected outcome |
|---|---|---|---|
| `custom-schema-registry.example` | `custom-widget` | `https://nirmalnr.github.io/crawler-test/schemas/custom-widget-registry.schema.json` (valid, in this repo) | `Verified` — ingested |
| `custom-schema-registry.example` | `custom-schema-unreachable` | `https://nirmalnr.github.io/crawler-test/schemas/does-not-exist.schema.json` (404) | Never reaches `verify.File` — `schemaCache.Content` fails first, logged as `schema_dereference_failed`; **skipped** |

This replaces an earlier, incorrect fixture (`dedi.schema-url-unresolved.json`, formerly in
`bad-registry-examples.example`) that reused one of the *canonical* schema URLs and so — once
actually crawled — verified and ingested just fine instead of demonstrating a failure.

## Registry removal (lifecycle test)

`lifecycle-test-registry.example` tests deletion, not just creation: `internal/reconcile.SyncDomain`
diffs each crawl's verified registries against DeDi.global's current list (`liveByName`) and deletes
whatever's no longer in the manifest, rather than leaving it orphaned forever. Confirmed as an actual
two-crawl sequence, not just read from source:

1. Manifest listed one registry, `temp-registry` (record: `temp-record-1`). Crawled — namespace,
   registry, and record all resolved via the lookup API (`record_count: 1`).
2. `temp-registry` removed from `files[]`, manifest re-signed, re-crawled. `temp-registry` and its
   record both now `404`; the namespace itself still resolves (`registry_count: 0`) — removing a
   registry doesn't delete its namespace, just that one registry.

The bad-registry-examples fixture history above hit this same path incidentally
(`dedi.schema-url-unresolved.json`'s replacement caused its old registry to get cleaned up the same
way) — this fixture isolates it as a deliberate, minimal, repeatable case.

### Record removal (same domain, three crawls)

The same domain then tested record-level add/remove — `internal/reconcile.syncRecords` diffs a
registry's desired records against its live ones on every crawl (a registry's file digest changing
is what makes this run at all; an unchanged digest short-circuits `syncRegistry` before records are
even looked at). Continuing from the state above:

1. `temp-registry` restored with one record, `temp-record-1`. Crawled — present (`200`).
   `temp-record-2` didn't exist yet (`404`), confirmed as this test's baseline.
2. `temp-record-2` added to the same file (new file digest), re-signed, re-crawled.
   `temp-record-2` now `200` (`record_count: 2`); `temp-record-1` unaffected.
3. `temp-record-2` removed again (back to just `temp-record-1`), re-signed, re-crawled.
   `temp-record-2` now `404` again (`record_count: 1`); `temp-record-1` still `200` throughout —
   confirming only the removed record was touched, not the whole registry.

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
| `dedi.invalid-schema.json` | The inline JSON Schema itself is malformed (`required` is a string, not an array) | `Rejected` / `schema_invalid` |
| `dedi.malformed.json` | The file isn't valid JSON (truncated) | `Rejected` / `unparseable` |

### Not covered here

Three `internal/verify` reasons aren't demonstrated by any fixture in this repo:

- **`domain_mismatch` / `manifest_domain_mismatch`** — require running *without*
  `allow_non_standard_domains`, which every fixture here depends on to be crawlable at all as a
  GitHub Pages path.
- **`canonicalization_failed`** — not practically constructible by hand-editing otherwise-valid JSON.
- **`schema_not_resolved`** — unreachable via a real crawl. `internal/reconcile.SyncDomain` always
  resolves a URL schema (fetch + compile) *before* calling `verify.File`, and `continue`s past the
  file entirely if that fails — `verify.File`'s own `schema == nil` branch that produces this reason
  only fires when something calls it directly, bypassing `reconcile`, the way earlier ad-hoc checks
  in this repo's history did.
