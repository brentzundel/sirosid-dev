# Phase 4 — Verification Checklist (manual, human-run)

No Docker or Flutter/Dart SDK exists in the sandboxed environment these
slices were built in, so **nothing in Slices A/B/C has been compiled, linted,
or run yet.** This checklist is for the human owner to run locally. Report
back what breaks — issues should be traceable to a specific slice/file via
the notes below.

## 1. Backend (Slice A)

```bash
cd sirosid-dev
make up VC=yes
# if already running: docker compose restart vc-apigw vc-verifier
make status
curl -s http://localhost:9003/.well-known/openid-credential-issuer | jq '.credential_configurations_supported.employee_id'
```
Expect a config block with `vct: "urn:yubico:employeeid:1"`, format
`dc+sd-jwt`. If missing/404: check `fixtures/vc-config.yaml`'s new
`common.credential_metadata.employee_id` / `apigw.data_sources.datastore.scopes.employee_id`
blocks, and that `fixtures/vc-metadata/vctm_employee_id.json` /
`fixtures/vc-bootstrapping/employee_id.json` are valid and bind-mounted.

Optionally, do a manual PAR+token+credential round trip (mirroring
`test-oidc-flow.mjs`/`tests/test_issuance_flows.py`) against `client_id:
yubiid-app`, `scope: employee_id` to confirm end-to-end before involving the
app at all.

## 2. Protocol client (Slice B)

```bash
cd hippo-yubico-app-user/packages/vc_wallet_client
dart pub get
dart analyze
dart test
```
If green, optionally un-skip `test/integration/live_stack_test.dart` against
the now-running stack from step 1 — this is the first real exercise of the
**highest-risk unverified assumption**: the `vp_token`/form-body shape POSTed
to `POST /verification/oidc-direct_post` in
`lib/src/presentation/presentation_service.dart` (no OpenAPI spec exists for
vc-verifier's endpoint, so this was reconstructed from report 01's prose
description, not a live capture).

Secondary risk to watch: `lib/src/issuance/issuance_service.dart`'s PAR call
sends no DPoP header (assumed unnecessary based on report 01's wording) — if
the live PAR endpoint 400s on this, that's the second place to check.

## 3. App (Slice C)

```bash
cd hippo-yubico-app-user
flutter pub get
flutter analyze
flutter test
```
Confirm the pre-existing IDV/passkey-manager test suites remain green
(nothing under `lib/features/enrollment|idv|credentials|yubikeys|hardware_key|software_passkey/**`
was touched, but this confirms it holds).

## 4. Manual end-to-end (device/simulator + the stack from step 1)

1. Launch the app → dashboard shows a new "Verifiable credentials" card
   (`VcWalletEntryCard`, inserted in `dashboard_home_page.dart`).
2. Obtain a credential-offer QR/link for `client_id: yubiid-app`,
   `scope: employee_id` from vc-apigw. Scan (in-app) or open (deep link,
   scheme `openid-credential-offer://`) it.
3. Expect: claims-preview consent screen (all 6 Employee ID claim types,
   no values yet) → accept → issuance progress → success → lands on the
   credential detail screen, and the credential now appears in wallet home.
4. Obtain an OpenID4VP presentation request for `employee_id` from
   vc-verifier (`/authorize?scope=employee_id&client_id=yubiid-app`, or
   whatever QR/deep-link it returns — scheme `openid4vp://`). Scan/open it
   from the wallet-home "Present a credential" button.
5. Expect: presentation-consent screen showing the verifier's client_id and
   every Employee ID claim (v1 always fully discloses) → accept → submit →
   result screen (may require "Check status again" if the RP's poll hasn't
   resolved yet).

## Known gaps, not bugs (by design, per the approved scope)

- No PAR/PKCE/authorization_code UI yet — only the pre-authorized_code path
  has screens. The port methods (`beginAuthorizationCodeIssuance`/
  `completeAuthorizationCodeIssuance`) and the `app.yubiid.vc://callback`
  redirect URI are fully wired on both platforms for this to be added later
  without further plumbing.
- mDL, PID, Diploma, EHIC: out of scope for this v1 entirely (Employee ID
  only, per product-owner decision).
- No hardware-backed (StrongBox/Secure Enclave/R2PS) keys — software P-256
  keys only (per product-owner decision).
- CI workflow secrets (`SIROSID_VC_*`) not yet added to
  `.github/workflows/*.yml` — `.env.local.example` was updated, CI wiring
  was left as a follow-up.

## Reference

Full design: `04-integration-design.md`. Implementation notes with complete
file lists: `05a-implementation-notes-backend.md` (Slice A),
`05b-implementation-notes-protocol-client.md` (Slice B),
`05c-implementation-notes-app.md` (Slice C).
