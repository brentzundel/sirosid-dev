# YubiId Verifiable-Credentials Wallet — Integration Design

Synthesizes reports 01–03 into a go/no-go design for adding a **brand-new,
purely additive** verifiable-credentials (VC) wallet capability to YubiId,
backed by the SIROS ID stack. Per product-owner direction, this does **not**
touch the existing YubicoId/IDV flow (Private ID-backed) or the FIDO2
passkey-manager features — those are separate concerns and out of scope
everywhere below. This document is the spec handed to Phase 3 implementation
agents; it intentionally contains no code.

**Revision note:** this version supersedes the original v1 scope (PID + mDL)
after product-owner review. v1 is now a single, purpose-built demo scenario:
**issue an Employee ID credential to the wallet, then present it** — a new
custom credential type, not one of sirosid's existing PID/mDL/Diploma/EHIC
fixtures. mDL is dropped entirely from v1 (issuance and presentation both).

---

## 1. Target functionality for v1

**Credential type: a new custom "Employee ID" credential**, not any of
sirosid's existing bootstrapped types. Defined fresh as an SD-JWT VC
(`dc+sd-jwt`, same mechanism PID already uses), since that's the simpler of
the two formats the stack supports (no CBOR/mso_mdoc parsing needed) and is a
natural fit for a corporate-badge-style credential.

**Claims (minimal set, per product-owner direction):**
- `employee_id` — unique identifier string
- `full_name`
- `department`
- `job_title`
- `photo` — base64-encoded image, same convention as PID's `picture` claim
- `issue_date`

**Suggested VCT identifier:** `urn:yubico:employeeid:1` (placeholder — Slice A
owner should confirm naming convention against sirosid's `urn:eudi:pid:1`
precedent before finalizing).

**Flows — both in scope for v1:**
- **OpenID4VCI issuance.** Build the **pre-authorized_code grant first**
  (credential offer → `/token` with
  `urn:ietf:params:oauth:grant-type:pre-authorized_code` → `/credential`): no
  PAR, no `/authorize` browser hand-off, no consent UI at the apigw layer —
  fastest path to an end-to-end demo. Add the full **PAR + PKCE +
  `authorization_code` flow with DPoP** as a fast-follow within the same v1
  effort once the pre-authorized path works.
- **OpenID4VP presentation** of the Employee ID credential — basic RP-initiated
  flow, DCQL, `direct_post`. Since this is a brand-new credential type, there's
  no pre-existing verifier-config gap to work around (unlike mDL) — the new
  VCT is added to `verifier.inbound.openid4vp.supported_credentials` as part
  of Slice A, alongside the issuance-side config, so presentation is a first-
  class part of v1 rather than blocked on anything pre-existing.

**Explicitly out of scope for v1:** mDL (dropped entirely — no mso_mdoc/CBOR
work needed anywhere in v1), PID/Diploma/EHIC (not needed for this demo
scenario), refresh-token flow, deferred-credential polling, WIA (not
implemented backend-side per report 01 §1), R2PS/WSCD hardware-backed
proof-of-possession keys (no client exists anywhere in scope — software
P-256 keys via `flutter_secure_storage`/platform keystore are sufficient for
Key Attestation, confirmed acceptable by product owner), credential
backup/sync via `go-wallet-backend`'s `/storage/vc/*`.

---

## 2. Where the OpenID4VCI/OpenID4VP protocol client lives

**Recommendation unchanged: pure Dart**, in a new local Dart package inside
the `hippo-yubico-app-user` repo (e.g. `packages/vc_wallet_client/`,
referenced via a `path:` pubspec dependency — not a new git repo). **Do not
add this to `hippo-yubico-sdk-flutter`.**

Justification (now simpler, since mDL/CBOR is out of scope entirely):
- **iOS has nothing to reuse.** Report 03 confirms iOS-side of the SDK plugin
  has no HTTP/crypto dependencies beyond the closed vendored xcframeworks —
  building the protocol client natively means 100% new Swift work with zero
  head start, identical in cost to writing it in Dart once.
- **Android's reusable deps don't save meaningful work.** Generic HTTP/crypto
  libraries have direct Dart equivalents (`dio`/`http`, `cryptography` —
  **already an app dependency** per `pubspec.yaml`, `dart_jsonwebtoken`/`jose`,
  `uuid`). Building once in Dart covers both platforms.
- **Avoids the SDK repo's release friction** (no CI, no publish, git-ref
  pinning per report 03 §5) — pure overhead for fast-iterating new protocol
  logic with no existing closed-SDK logic being reused here.
- **DPoP, PAR, JWT proof-of-possession, SD-JWT VC parsing are all standard,
  well-precedented pure-Dart operations.** With mDL dropped, **no CBOR/COSE
  library is needed at all for v1** — this simplifies Slice B's dependency
  list further than originally scoped.

**Net effect: zero changes to `hippo-yubico-sdk-flutter`** for v1.

---

## 3. New app-side surface

New feature area, following the app's existing feature-folder convention
(`domain/` / `state/` / `presentation/` / `navigation/`, as seen in
`lib/features/credential_setup/` and `lib/features/software_passkey/`):

```
lib/features/verifiable_credentials/
  domain/        # CredentialOffer, IssuedCredential, PresentationRequest,
                 # ConsentClaim, VcEnvironment value types
  state/         # Riverpod controllers: issuance flow, presentation flow,
                 # wallet-home (credential list) state
  presentation/  # screens (below)
  navigation/    # route registration + flow-routing glue, mirroring
                 # credential_setup/navigation/
```

**Screens/states needed:**
1. **Offer entry** — QR scan (reuse `mobile_scanner`, already a dependency)
   and/or deep-link handling for a sirosid-issued credential-offer URI,
   resolving via `GET /credential-offer/:uuid` or an inline offer payload.
2. **Issuance progress** — token/DPoP-nonce-retry/credential-fetch progress;
   needs real intermediate states (unlike the existing
   `IdvYubicoIdFinalizingScreen`'s single opaque await, per report 02 §4).
3. **Consent / claims-preview screen — new concept, does not exist anywhere
   in the app today** (report 02 §4 confirms this explicitly). Shown before
   finalizing issuance and, critically, before OpenID4VP presentation —
   displays the Employee ID claims (name/department/title/photo) about to be
   shared.
4. **Wallet home (credential list)** — the new wallet's main surface; for v1,
   lists the single issued Employee ID credential (designed to generalize to
   multiple credentials later, but only one type exists in v1).
5. **Credential detail view** — renders the Employee ID's disclosed SD-JWT
   claims (name, department, title, photo, employee_id, issue_date).
6. **Presentation entry** — QR/deep-link resolution of an `openid4vp://`
   request URI, fetching the signed request object.
7. **Presentation consent screen** — what's being shared, with whom (verifier
   identity/trust status), accept/decline.
8. **Presentation result screen** — success/failure after `POST
   /verification/oidc-direct_post`.

**New ports:** a new port file,
`lib/app/infrastructure/verifiable_credentials/vc_wallet_ports.dart`, kept
structurally parallel to (but never importing/modifying) `yubico_sdk_ports.dart`:

```dart
abstract class VerifiableCredentialSdkPort {
  Future<CredentialOffer> resolveCredentialOffer(String uriOrPayload);
  Future<IssuedCredential> issueCredential({
    required CredentialOffer offer,
    required String credentialConfigurationId,
  });
  Future<List<StoredVerifiableCredential>> listStoredCredentials();
  Future<StoredVerifiableCredential> getStoredCredential(String credentialId);
  Future<void> deleteStoredCredential(String credentialId);

  Future<PresentationRequest> resolvePresentationRequest(String requestUri);
  Future<PresentationResult> submitPresentation({
    required PresentationRequest request,
    required String credentialId,
    required Set<String> consentedClaimPaths,
  });
}
```
Explicitly **not** merged into `YubicoIdSdkPort`, `CredentialsSdkPort`,
`HardwareKeySdkPort`, or `SoftwarePasskeySdkPort` — those remain untouched.

**Storage:** issued credentials persisted locally via `flutter_secure_storage`
(already a dependency), keyed by `credential_identifier`.

---

## 4. Sirosid-side changes needed

Larger than originally scoped, since Employee ID is a **brand-new credential
type**, not a relabeled existing one — but still entirely config/fixture-data,
**no Go source changes** anticipated:

- **New VCTM metadata file**: `sirosid-dev/fixtures/vc-metadata/vctm_employee_id.json`,
  modeled on the existing `vctm_pid_arf_1_8.json`, declaring
  `vct: "urn:yubico:employeeid:1"` and a claims schema for `employee_id`,
  `full_name`, `department`, `job_title`, `photo`, `issue_date`.
- **New bootstrapping fixture**: `sirosid-dev/fixtures/vc-bootstrapping/employee_id.json`,
  modeled on `pid_1_8.json`'s structure, with at least one sample employee
  record (can reuse the existing `alice-001`/`bob-002`/`carol-003` test-user
  keys and `identity_mappings.json` wiring, or introduce a new fictional
  employee — Slice A owner's call, either works).
- **`vc-config.yaml` additions:**
  - `common.credential_metadata`: new `employee_id` entry, `format: "dc+sd-jwt"`,
    `vctm_file_path: "/metadata/vctm_employee_id.json"`.
  - `apigw.data_sources.datastore.scopes`: new `employee_id` scope entry
    (`auth_provider: oidc`, `auth_claims: ["given_name","family_name"]` — or
    simpler, since there's no PID-style mandatory-expiry-claim complication),
    `import.file_paths: ["/bootstrapping/employee_id.json"]`.
  - `apigw.delivery.openid4vci.clients`: new client entry, e.g.:
    ```yaml
    "yubiid-app":
      type: "public"
      redirect_uri: "app.yubiid.vc://callback"   # must not collide with the
                                                   # existing "siros-sample://"
                                                   # test-wallet scheme
      scopes: ["employee_id"]
    ```
  - `apigw.delivery.credential_offers.wallets`: matching new wallet entry
    (label + same `redirect_uri`) for credential-offer QR generation.
  - `verifier.inbound.openid4vp.supported_credentials`: new entry for the
    `urn:yubico:employeeid:1` VCT (this is what makes presentation possible —
    since it's a new type, there's no pre-existing gap to work around, just a
    new addition alongside the others).
  - `verifier.inbound.openid4vp.clients`: new client entry scoped to
    `employee_id`, mirroring `e2e-test-client`'s shape.
- **WebAuthn passkey registration — deferred, no `go-wallet-backend` changes
  for v1.** Confirmed unnecessary: pre-authorized_code needs no wallet-side
  auth; `go-wallet-backend` is not in the v1 request path at all — the wallet
  talks directly to `vc-apigw`/`vc-verifier`, exactly as the existing Go CLI
  test client (`vc`'s `internal/wallet/apiv1/{vci,vp}.go`) does.
- **Trust PDP stays transparent** (allow-all by default in dev, per existing
  config) — no changes needed.
- **One legitimate optional callout (non-blocking):** PDP-deny errors in
  `vc`'s `pkg/trust/gotrust.go` surface as plain `fmt.Errorf` → HTTP 500
  instead of a typed 401/403 (report 01 §4) — worth a separate fast-follow
  fix, not required to ship v1.
- **R2PS/WSCD:** no backend wiring needed for v1 (software keys only,
  confirmed acceptable by product owner).

---

## 5. Vertical slices for Phase 3 (3 slices, bounded by repo/directory)

### Slice A — sirosid-dev config + new Employee ID credential type
- **Repo/files:** `sirosid-dev/fixtures/vc-metadata/vctm_employee_id.json`
  (new), `sirosid-dev/fixtures/vc-bootstrapping/employee_id.json` (new),
  `sirosid-dev/fixtures/vc-config.yaml` (edits per §4: credential_metadata,
  datastore scope, openid4vci client, credential_offers wallet,
  openid4vp supported_credentials + client). No other repo touched.
- **Acceptance criteria:** `make up VC=yes` starts with the new
  `employee_id` scope and `yubiid-app` client present; `GET
  /.well-known/openid-credential-issuer` lists `employee_id` under
  `credential_configurations_supported`; a manual `curl`/Go-CLI-test-client
  pre-authorized_code → token → credential round trip succeeds for
  `employee_id` and returns a valid SD-JWT VC with the expected claims;
  `vc-verifier`'s `/authorize?scope=employee_id` accepts the new client_id
  and returns a `deep_link_url` referencing the new VCT.

### Slice B — protocol client package (pure Dart, no UI)
- **Repo/files:** `hippo-yubico-app-user/packages/vc_wallet_client/` (new
  local package: `pubspec.yaml`, `lib/vc_wallet_client.dart` public API,
  `lib/src/{issuance,presentation,jose,dpop,models}/*.dart`, `test/**`).
  Root `hippo-yubico-app-user/pubspec.yaml` gets a `path:` dependency on this
  package plus net-new deps (`dio` or `http`, a JOSE/JWT library, `uuid` —
  `cryptography` is already present; **no CBOR/mso_mdoc library needed**,
  since mDL is out of scope). **No changes to `lib/features/**` or
  `lib/app/**` in this slice.**
- **Acceptance criteria:** package-level tests exercise, against Slice A's
  locally-running stack: (1) pre-authorized_code grant → `/token` →
  `/credential` for `employee_id`, including DPoP proof generation and nonce
  retry, successfully parsing the returned SD-JWT VC and its disclosed
  claims; (2) PAR + PKCE + `authorization_code` + DPoP round trip; (3)
  OpenID4VP: resolve a request object, build a DCQL-matching `vp_token` for
  `employee_id`, `POST /verification/oidc-direct_post`, observe success via
  `/poll/:session_id`. All exercised via package tests/CLI, no app UI
  involved.

### Slice C — app feature UI, ports, navigation, config
- **Repo/files:** `hippo-yubico-app-user/lib/app/infrastructure/
  verifiable_credentials/vc_wallet_ports.dart` (new port(s) per §3, plus a
  Riverpod provider wiring the concrete implementation to Slice B's package);
  `lib/features/verifiable_credentials/**` (all screens/state/navigation per
  §3); `lib/app/config/app_sirosid_vc_config.dart` (new, analogous to
  `app_idv_session_config.dart`); router registration; `.env.local.example`
  additions (§6); Android manifest / iOS `Info.plist` custom-scheme
  registration for the chosen VC redirect URI. **Does not touch
  `YubicoIdSdkPort`, `CredentialsSdkPort`, `HardwareKeySdkPort`,
  `SoftwarePasskeySdkPort`, or any file under `lib/features/enrollment/**`,
  `lib/features/idv/**`, `lib/features/credentials/**`,
  `lib/features/yubikeys/**`, `lib/features/hardware_key/**`,
  `lib/features/software_passkey/**`.**
- **Acceptance criteria:** from a cold app launch, scanning/opening a sirosid
  credential-offer QR/deep link completes an OpenID4VCI pre-authorized_code
  flow against the locally-running Slice A stack and displays the issued
  Employee ID credential (name/department/title/photo) in the new
  credential-list ("wallet home") screen and its detail view; the user can
  initiate presentation of the Employee ID credential against a scanned
  OpenID4VP request, see the new consent screen showing the claims to be
  shared, and complete presentation successfully, landing on a success
  result screen. Existing IDV/passkey-manager test suites (report 02 §6,
  ~4,400 lines) remain green and untouched.

---

## 6. Config/env additions

Following the exact `PRIVATE_ID_*` pattern in `.env.local.example` /
`AppIdvSessionConfig.fromEnvironment()`:

**New `.env.local.example` keys:**
```
SIROSID_VC_APIGW_BASE_URL=
SIROSID_VC_VERIFIER_BASE_URL=
SIROSID_VC_CLIENT_ID=
SIROSID_VC_REDIRECT_URI=
SIROSID_VC_ENVIRONMENT=
```

**New Dart config class:** `lib/app/config/app_sirosid_vc_config.dart` —
`AppSirosIdVcConfig.fromEnvironment()` reading the above via
`String.fromEnvironment`, with a `missingFields()` fail-fast helper matching
`AppIdvSessionConfig`'s convention.

**CI:** add `SIROSID_VC_*` secrets to both `.github/workflows/android-ci.yml`
(warn-and-continue on missing, matching current `PRIVATE_ID_*` handling) and
`.github/workflows/android-yubico-brand-release.yml` (hard-fail on missing).

**Environment naming:** `SIROSID_VC_ENVIRONMENT` should be its own
independent value, not forced into the existing YubicoId/Private-ID flow's
`{demo, dev, stage, prod}` enum, since the VC wallet's deployment axis
doesn't necessarily line up with YubicoId's.

**Custom URL schemes (Slice C, both platforms):** register the app's
`SIROSID_VC_REDIRECT_URI` scheme (e.g. `app.yubiid.vc://callback`) as an
Android intent-filter and an iOS `CFBundleURLTypes` entry — must match
whatever Slice A configures in `vc-config.yaml`'s `redirect_uri` for the new
client, and must not collide with the existing `siros-sample://` test-wallet
scheme.

---

## Summary for go/no-go

- **v1 = a single new custom credential type, "Employee ID"** (SD-JWT VC),
  issued and then presented — not PID/mDL as originally scoped. Confirmed
  with product owner.
- **mDL is dropped entirely from v1** — no CBOR/mso_mdoc work anywhere,
  simplifying Slice B's dependencies.
- **No blocking dependency on the opaque `hippo-yubico-sdk-flutter` native
  SDK** — this design avoids it entirely (§2).
- **No `go-wallet-backend`/`go-trust`/`vc` source changes required** — only
  new YAML config + two new fixture/metadata files in `sirosid-dev` (§4).
- **Software (non-hardware-backed) proof-of-possession keys are acceptable
  for v1** — confirmed with product owner.
- **Three cleanly-bounded slices** (§5): sirosid config + new credential type,
  a pure-Dart protocol client package, and app UI/ports/navigation —
  overlapping only on one `pubspec.yaml` edit.
- **Issuance phasing**: pre-authorized_code first, full PAR/authorization_code
  /DPoP flow as a fast-follow within the same v1 effort (not contradicted by
  product owner; carried over from the original recommendation).
