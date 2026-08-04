# Slice A — sirosid-dev config + new "Employee ID" credential type

Implementation notes for Slice A of the YubiId VC wallet integration effort
(see `docs/integration/04-integration-design.md` §1, §4, §5 Slice A). This
slice is config/fixture-only, entirely within `sirosid-dev`. No other repo
(`go-wallet-backend`, `go-trust`, `vc`) was touched, and no Go source was
changed.

## Files created

- `fixtures/vc-metadata/vctm_employee_id.json` — new SD-JWT VC Type Metadata
  (VCTM) document, modeled on `vctm_pid_arf_1_8.json`. Declares
  `vct: "urn:yubico:employeeid:1"` and a claims schema for `employee_id`
  (string, mandatory), `full_name` (string, mandatory), `department` (string,
  mandatory), `job_title` (string, mandatory), `photo` (string,
  base64-encoded image, same convention as PID's `picture` claim, not
  mandatory), and `issue_date` (string, mandatory). Includes `display`
  metadata (`name: "Employee ID"`, English locale, a placeholder `rendering`
  logo reusing the same demo SVG asset the PID VCTM points at — swap for a
  real Employee ID badge asset before this goes beyond dev/demo use).

- `fixtures/vc-bootstrapping/employee_id.json` — new bootstrapping fixture,
  modeled on `pid_1_8.json`'s structure (one JSON object keyed by test-user
  id, `meta.scope: "employee_id"`, `document_data` with the claim values).
  Populated three sample records, reusing the **existing** mini-oidc test
  users already defined in `fixtures/vc-bootstrapping/identity_mappings.json`
  (`alice-001`, `bob-002`, `carol-003`) — see "Identity mapping decision"
  below for why no new mapping entry was added.
  - `alice-001` → `employee_id: "EMP-1001"`, `full_name: "Helen Mirren"`,
    `department: "Engineering"`, `job_title: "Software Engineer"`,
    `issue_date: "2026-01-15"`, `photo`: the same placeholder base64 PNG
    already used by the PID fixtures for `picture`.
  - `bob-002` → `EMP-1002`, "Jason Momoa", "Product Design", "Senior Product
    Designer", `2026-02-01`, a short placeholder base64 string for `photo`.
  - `carol-003` → `EMP-1003`, "Gary Oldman", "Customer Success", "Support
    Engineer", `2026-03-10`, same short placeholder for `photo`.

  Only one record is strictly required by the acceptance criteria; all three
  were added for parity with `pid_1_8.json`/`diploma.json` (lets any of the
  three mini-oidc test users exercise the `employee_id` scope, useful for
  Slice B/C testing without being tied to a single login).

## Files edited

- `fixtures/vc-config.yaml` — six additions, all additive (no existing keys
  changed):
  1. `common.credential_metadata.employee_id`: `format: "dc+sd-jwt"`,
     `vctm_file_path: "/metadata/vctm_employee_id.json"`.
  2. `apigw.data_sources.datastore.scopes.employee_id`: `auth_provider: oidc`,
     `auth_claims: ["given_name", "family_name"]` — same shape as
     `pid_1_8`/`diploma`. Also added
     `/bootstrapping/employee_id.json` to `apigw.data_sources.datastore.import.file_paths`.
  3. `apigw.delivery.openid4vci.clients."yubiid-app"`: `type: "public"`,
     `redirect_uri: "app.yubiid.vc://callback"`, `scopes: ["employee_id"]`.
  4. `apigw.delivery.credential_offers.wallets.yubiid`: `label: "YubiId"`,
     `redirect_uri: "app.yubiid.vc://callback"` (same URI as the client
     above, for credential-offer QR generation).
  5. `verifier.inbound.openid4vp.supported_credentials`: new entry
     `vct: "urn:yubico:employeeid:1"`, `scopes: ["employee_id"]`.
  6. `verifier.inbound.openid4vp.clients."yubiid-app"`: `type: "public"`,
     `redirect_uri: "app.yubiid.vc://callback"`, `scopes: ["employee_id"]` —
     mirrors `e2e-test-client`'s shape, scoped only to `employee_id`.

## Exact values for Slices B/C to match

These are the load-bearing identifiers the Dart protocol-client package
(Slice B) and the app UI (Slice C) must use verbatim to interoperate with
this config:

| Item | Value |
|---|---|
| VCT (credential type) | `urn:yubico:employeeid:1` |
| Scope name | `employee_id` |
| OpenID4VCI client_id | `yubiid-app` |
| OpenID4VP client_id | `yubiid-app` |
| Redirect URI (both VCI and VP clients, and the wallet entry) | `app.yubiid.vc://callback` |
| Wallet label (credential-offer UI) | `YubiId` (key `yubiid` in `credential_offers.wallets`) |
| Claims | `employee_id`, `full_name`, `department`, `job_title`, `photo`, `issue_date` (all top-level, no nested objects) |
| Credential format | `dc+sd-jwt` |

**Redirect URI collision check performed:** grepped `fixtures/vc-config.yaml`
for every existing `redirect_uri` value before choosing
`app.yubiid.vc://callback`. Existing values in the file are
`siros-sample://callback`, `http://localhost:3000/cb`, and
`http://localhost:3000/callback` — no collision with the design doc's
suggested `app.yubiid.vc://callback`, so it was used as-is (no scheme change
needed).

## Identity mapping decision

Chose to **reuse the existing `alice-001`/`bob-002`/`carol-003` test-user
keys** from `identity_mappings.json` rather than adding a new identity
mapping entry. Rationale:
- The `employee_id` datastore scope (like `pid_1_8`/`diploma`/`mdl`) resolves
  via `auth_provider: oidc` against mini-oidc, keyed by
  `identity_mapping_ids`. Those three users are already wired end-to-end
  (mini-oidc login → identity mapping → datastore document lookup), so no
  new mini-oidc user or mapping plumbing was needed — this is the simplest
  option.
- Adding a fourth identity (e.g. a fictional "dave-004") would require also
  extending `identity_mappings.json` and confirming mini-oidc's user list
  includes a matching login, for zero added test coverage over reusing
  `alice-001`.
- This mirrors exactly how `pid_1_8.json`, `diploma.json`, and `mdl.json`
  are bootstrapped today (§4 of the design doc explicitly calls this out as
  an acceptable option).

`fixtures/vc-bootstrapping/identity_mappings.json` was **not modified**.

## Verification status

**Not run live — Docker is not available in this environment** (`docker`,
`node`, and `python3` were all absent from the sandboxed shell used for this
work; only `git`/`bash` core tooling was present). No container build/start
was attempted, so the acceptance criteria in design-doc §5 Slice A have not
been empirically confirmed.

What was checked instead, statically:
- Both new JSON files were re-read and are well-formed (balanced
  braces/brackets, valid string escaping) — hand-verified against the
  templates they were modeled on.
- `fixtures/vc-config.yaml` was re-read in full after edits; all six
  additions are structurally parallel to existing sibling entries
  (`pid_1_8`, `diploma`, `e2e-test-client`), and no existing key was
  altered.
- Confirmed via `docker-compose.vc-services.yml` that `./fixtures/vc-metadata`
  and `./fixtures/vc-bootstrapping` are bind-mounted read-only into the
  `vc-apigw` container at `/metadata` and `/bootstrapping` respectively, and
  `./fixtures/vc-config.yaml` is mounted at `/config.yaml` — so the new
  `vctm_file_path`/`import.file_paths` values in the YAML resolve correctly
  inside the container without any path changes.
- Confirmed via `Makefile` (`_HOST := $(if $(DOMAIN),$(DOMAIN),localhost)`
  and `VC_APIGW_URL ?= http://$(_HOST):9003`) and
  `docker-compose.vc-services.yml` (`vc-apigw` maps host port `9003` →
  container port `8080`) that, once the stack is up, the apigw's
  OpenID4VCI issuer metadata is reachable at
  `http://localhost:9003/.well-known/openid-credential-issuer`.

**Manual verification steps for a human with Docker available**, from
`sirosid-dev/`:

```sh
make up VC=yes
# wait for health checks to pass, then:
curl -s http://localhost:9003/.well-known/openid-credential-issuer | jq '.credential_configurations_supported.employee_id'
# expect a non-null object with format "dc+sd-jwt" and the vctm claims schema

curl -s http://localhost:9001/.well-known/openid-configuration | jq .
# (verifier OIDC discovery; supported_credentials/clients aren't exposed via
# a well-known endpoint - confirm the yubiid-app client + employee_id VCT via
# the vc-verifier logs or an actual /authorize?scope=employee_id round trip)

# Full pre-authorized_code -> token -> credential round trip (per design doc
# §1 and Slice A acceptance criteria) is most easily driven via the existing
# Go CLI test client in the `vc` repo (internal/wallet/apiv1/vci.go) pointed
# at client_id=yubiid-app, scope=employee_id, redirect_uri=
# app.yubiid.vc://callback against http://localhost:9003.
```

If `make up VC=yes` fails to start or the `employee_id` key is missing from
`credential_configurations_supported`, the most likely causes to check first
are: the two new fixture files not being present under `fixtures/` at
container-build/mount time (confirm bind mounts picked up the new files —
may need `make down` / `make up` rather than a live reload), or a YAML
indentation issue in `vc-config.yaml` (re-diff against the `pid_1_8`/
`diploma` sibling blocks if so).

## Notes / deviations from the design doc

- None. All six `vc-config.yaml` additions, both new fixture files, the
  chosen VCT, scope name, client_id, and redirect URI match §4/§5 of
  `04-integration-design.md` exactly.
