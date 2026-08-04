# Slice C — app feature UI, ports, navigation, config

Implementation notes for Slice C of the YubiId VC wallet integration effort
(see `docs/integration/04-integration-design.md` §1, §3, §5 Slice C, §6).
Everything lives in the `hippo-yubico-app-user` repo. This slice builds the
`lib/features/verifiable_credentials/` feature (UI, state, navigation) and
its supporting port/config, wiring the app to Slice B's `vc_wallet_client`
package (`docs/integration/05b-implementation-notes-protocol-client.md`) so a
user can scan/open a credential offer, complete OpenID4VCI
pre-authorized_code issuance of the Employee ID credential, see it in a
wallet-home list + detail view, scan/open an OpenID4VP presentation request,
see a consent screen, and complete presentation, landing on a result screen.

No file under `lib/features/enrollment/**`, `lib/features/idv/**`,
`lib/features/credentials/**`, `lib/features/yubikeys/**`,
`lib/features/hardware_key/**`, `lib/features/software_passkey/**` was
touched, and `YubicoIdSdkPort`, `CredentialsSdkPort`, `HardwareKeySdkPort`,
`SoftwarePasskeySdkPort`, and `yubico_sdk_service.dart` were not touched.

## Files created

**Port + config (`lib/app/**`):**
- `lib/app/infrastructure/verifiable_credentials/vc_wallet_ports.dart` —
  `VerifiableCredentialSdkPort` (abstract) + `VcWalletSdkService` (concrete),
  kept structurally parallel to (never importing/modifying)
  `yubico_sdk_ports.dart`.
- `lib/app/config/app_sirosid_vc_config.dart` — `AppSirosIdVcConfig`, mirroring
  `AppIdvSessionConfig`'s exact pattern (private constructor,
  `fromEnvironment()`, `missingFields()`, plus a Riverpod `Provider`).
- `lib/app/providers/vc_wallet_sdk_provider.dart` — wires
  `VerifiableCredentialSdkPort` to `VcWalletSdkService`, mirroring
  `yubico_sdk_provider.dart`'s wiring style.

**Feature (`lib/features/verifiable_credentials/`):**
- `domain/stored_verifiable_credential.dart` — the persisted-credential value
  type (JSON round-trip, `fromIssuedCredential`, `toIssuedCredential` which
  re-parses the raw SD-JWT via `VcWalletClient.parseSdJwtVc` since
  `IssuedCredential` itself has no `fromJson`/re-parse helper by design).
- `domain/vc_deep_link.dart` — classifies an incoming deep link as
  `credentialOffer` / `presentationRequest` / `authorizationRedirect` (or
  unrelated), mirroring `RemoteEnrollmentLink`/`ByoIdvCallbackLink`'s
  dependency-free domain-parsing style. Does *not* parse offer/request
  payload contents itself — those are handed to `vc_wallet_client` as-is.
- `domain/consent_claim.dart` — `ConsentClaim` display value type +
  `previewForIssuance()` (claim types only, values unknown pre-issuance) /
  `fromClaims()` (canonical order + values, for the presentation-consent and
  credential-detail screens).
- `state/vc_wallet_home_controller.dart` — `AsyncNotifier` backing the
  wallet-home list (`listStoredCredentials`), plus `vcStoredCredentialProvider`
  (`FutureProvider.family`) for the detail screen.
- `state/vc_issuance_flow_controller.dart` + `vc_issuance_flow_state.dart` —
  offer resolution → claims-preview/consent → (tx-code prompt if required) →
  issuance → store → success, as a `Notifier` over a sealed state hierarchy.
- `state/vc_presentation_flow_controller.dart` + `vc_presentation_flow_state.dart`
  — request resolution → consent → submit → best-effort poll → result.
- `presentation/vc_offer_entry_screen.dart`, `vc_presentation_entry_screen.dart`
  — QR-scan entry points (`mobile_scanner`, already a dependency), simplified
  relative to `EnrollmentQrCodeScreen` since offer/request parsing is fully
  delegated to `vc_wallet_client`.
- `presentation/vc_issuance_consent_screen.dart`,
  `vc_issuance_progress_screen.dart`, `vc_wallet_home_screen.dart`,
  `vc_credential_detail_screen.dart`, `vc_presentation_consent_screen.dart`,
  `vc_presentation_result_screen.dart` — the remaining screens (see
  "Screen-by-screen flow" below).
- `presentation/widgets/vc_claim_list.dart` — shared claim-rendering widget
  (used by issuance-consent, presentation-consent, and detail screens);
  special-cases the `photo` claim to attempt a base64-decoded image render,
  falling back to a generic icon if decoding fails (Slice A's bob/carol
  fixture photo values are short placeholders, not real images).
- `presentation/widgets/vc_wallet_entry_card.dart` — the dashboard entry
  point card (see "Dashboard wiring" below).
- `navigation/vc_wallet_routes.dart` — location-builder functions over
  `AppRoutes` path constants, mirroring `credential_setup_flow_routes.dart`.
- `navigation/vc_deep_link_listener.dart` — a second, independent deep-link
  listener widget (see "Deep-link wiring" below).

**Tests** (`test/features/verifiable_credentials/**`, `test/app/config/
app_sirosid_vc_config_test.dart`, `test/app/infrastructure/
verifiable_credentials/vc_wallet_ports_test.dart`,
`test/support/fake_vc_wallet_data.dart`, `test/support/
fake_verifiable_credential_sdk_port.dart`) — see "Test coverage" below.

## Files edited

- `lib/app/navigation/app_routes.dart` — added `vcWalletHome`,
  `vcCredentialDetailsRoot`/`Segment`, `vcOfferEntry`, `vcIssuanceConsent`,
  `vcIssuanceProgress`, `vcPresentationEntry`, `vcPresentationConsent`,
  `vcPresentationResult` path constants + `vcCredentialDetailLocation()`.
  No existing constant changed.
- `lib/app/navigation/app_router.dart` — registered 6 new standalone
  top-level `GoRoute`s (scan/consent/progress/result screens, mirroring how
  `enrollmentQrCode` also isn't nested under any shell) plus `vcWalletHome`
  (with a nested credential-detail sub-route) inside the existing
  `SessionTimeoutGuard` `ShellRoute`, as a sibling of `dashboardMore`/
  `account`/`settings`. No existing route's `builder`/`redirect` changed.
- `lib/app/app.dart` — wraps the existing `RemoteEnrollmentLinkListener` with
  a new `VcDeepLinkListener` (additive nesting, no existing behavior
  changed).
- `lib/features/dashboard/presentation/dashboard_home_page.dart` — inserted
  one `VcWalletEntryCard()` between the pending-credentials section and
  "Available credentials" (see "Dashboard wiring").
- `.env.local.example` — added the five `SIROSID_VC_*` keys with a comment
  block matching the `PRIVATE_ID_*` block's style, documenting Slice A's
  actual example values.
- `android/app/src/main/AndroidManifest.xml` — three new `<intent-filter>`
  blocks (scheme `app.yubiid.vc`/host `callback`; scheme
  `openid-credential-offer`; scheme `openid4vp`), sibling to the existing
  `yubicosoftkeys://` filter.
- `ios/Runner/Info.plist` — one new `CFBundleURLTypes` dict entry
  (`app.yubiid.vc`, `openid-credential-offer`, `openid4vp`), sibling to the
  existing `yubicosoftkeys`/`com.yubico.yaas` entry.
- `pubspec.yaml` — added `http: ^1.2.2` as a **direct** dependency (see
  "Deviation: added `http`" below). Slice B's `vc_wallet_client: path:
  packages/vc_wallet_client` line was not touched.

## How the port wires to Slice B's package

`VcWalletSdkService` lazily creates one `VcWalletClient` instance
(`VcWalletClient.create(config: ..., httpClient: ...)`) per service
lifetime (i.e. per app session, matching the package's "one instance == one
DPoP session" contract) and is otherwise a thin pass-through:

| Port method | `VcWalletClient` method |
|---|---|
| `resolveCredentialOffer` | `resolveOffer` |
| `issueCredentialPreAuthorized` | `issuePreAuthorized` |
| `beginAuthorizationCodeIssuance` | `beginAuthorizationCodeIssuance` (exposed, no screen built — see below) |
| `completeAuthorizationCodeIssuance` | `completeAuthorizationCodeIssuance` (same) |
| `resolvePresentationRequest` | `resolvePresentationRequest` |
| `submitPresentation` | `submitPresentation` (after `StoredVerifiableCredential.toIssuedCredential()` re-parses the raw SD-JWT) |
| `pollPresentationResult` | `pollPresentationResult` |

Storage (`storeIssuedCredential`/`listStoredCredentials`/
`getStoredCredential`/`deleteStoredCredential`) is Slice C's own addition on
top of the package, backed by `flutter_secure_storage`, keyed by
`credential_configuration_id` (v1 has exactly one credential type,
`employee_id`, so this doubles as a stable per-credential id; the shape
generalizes to multiple credentials later without change). The whole wallet
is persisted as one JSON map under a single secure-storage key
(`verifiable_credentials_wallet`), mirroring
`SecurePendingCredentialRequestStore`'s pattern exactly (including its
corrupted-payload-is-treated-as-empty defensive decode).

`VcWalletSdkService`'s constructor accepts an optional `httpClient`
parameter (forwarded to `VcWalletClient.create`), purely for test injection
(`package:http/testing.dart`'s `MockClient`) — production callers (the
`verifiableCredentialSdkPortProvider` wiring) never pass one.

## Screen-by-screen flow

**Issuance:**
1. `VcOfferEntryScreen` (QR scan, `mobile_scanner`) or a deep link → raw
   string handed directly to `VcIssuanceFlowController.resolveOffer` (no
   app-side parsing; `vc_wallet_client` accepts inline JSON,
   `credential_offer`/`credential_offer_uri` deep-link params, or a direct
   offer URL itself).
2. `VcIssuanceConsentScreen` — renders based on the controller's sealed
   state: claims-preview (all 6 Employee ID claim *types*, values unknown
   pre-issuance — the "new concept" screen called out in report 02 §4 as not
   existing anywhere in the app before this feature) → accept → if the
   offer's grant requires a `tx_code`, a prompt is shown inline (same
   screen, different state) → `confirmAndIssue()`.
3. `VcIssuanceProgressScreen` — real intermediate states (in progress /
   success / tx-code retry / error), unlike `IdvYubicoIdFinalizingScreen`'s
   single opaque await (per report 02 §4). On success, `pushReplacement`s to
   the credential-detail screen; on tx-code-retry/error, pops back to the
   consent screen.
4. `VcWalletHomeScreen` — credential list (v1: 0 or 1 item), "Add
   credential" FAB → offer entry, "Present a credential" button →
   presentation entry.
5. `VcCredentialDetailScreen` — renders all disclosed claims (via
   `ConsentClaim.fromClaims`/`VcClaimList`, including a best-effort
   base64-decoded photo render) + a "Remove from this device" action.

**Presentation:**
6. `VcPresentationEntryScreen` (QR scan / `openid4vp://` deep link) → raw
   string handed to `VcPresentationFlowController.resolveRequest`.
7. `VcPresentationConsentScreen` — shows the verifier's `client_id` and every
   claim of the (single, v1) stored credential that will be shared (v1's
   `vc_wallet_client` always fully re-discloses — no holder-side selective
   disclosure — so "claims to be shared" is simply the credential's full
   claim set) → accept → `confirmAndSubmit()`.
8. `VcPresentationResultScreen` — status-specific messaging
   (accepted/submitted/pending/failed) with a manual "Check status again"
   action for the polling cases, since `oidc-direct_post`-style flows
   typically still require polling to observe the RP's final outcome.

## Deep-link / URL-scheme wiring

Three schemes are registered (Android intent-filters + iOS
`CFBundleURLTypes`), all siblings of the existing `yubicosoftkeys://` entry,
none colliding with it or with `siros-sample://`:
- `app.yubiid.vc` / host `callback` — the default `SIROSID_VC_REDIRECT_URI`
  (matches Slice A's `yubiid-app` client config exactly). Only actually
  exercised by the PAR/PKCE/authorization_code fast-follow flow (not built
  in this slice — see "Deviations" below) but registered end-to-end now so
  nothing is left to wire up later.
- `openid-credential-offer` (any host) — OpenID4VCI credential-offer deep
  links opened outside the app (e.g. from an email/browser link, as opposed
  to a QR scan inside the app).
- `openid4vp` (any host) — OpenID4VP presentation-request deep links.

`VcDeepLinkListener` (`lib/features/verifiable_credentials/navigation/
vc_deep_link_listener.dart`) subscribes to the *same*
`deepLinkServiceProvider` stream `RemoteEnrollmentLinkListener` already
uses — deliberately a separate widget rather than adding branches to that
existing, tested listener, since the underlying stream is a broadcast stream
and supports independent listeners. It uses `VcDeepLink.tryParse` to
classify the link and, on a match, drives the appropriate flow controller
directly (`resolveOffer`/`resolveRequest`) before pushing to the matching
consent screen, exactly like the offer/presentation entry screens do after a
QR scan. Wired into the app tree in `app.dart` as
`RemoteEnrollmentLinkListener(child: VcDeepLinkListener(child: ...))` —
purely additive nesting.

## Dashboard wiring (making the feature reachable)

`VcWalletEntryCard` — a small tappable card ("Verifiable credentials / View
or add your Employee ID credential") — is inserted into
`DashboardHomePage`'s existing scroll view, between the pending-credentials
section and "Available credentials". This was chosen over adding a new
button to `TopBar` (whose existing `+` button is specifically wired to
credential-setup QR scanning, a different concern) as the least invasive way
to surface a genuinely new top-level destination without altering any
existing widget's behavior. Tapping it navigates to `/wallet/home`
(`VcWalletHomeScreen`), which is the actual entry point into every screen
listed above.

## Deviations from the design doc, and why

1. **Port method names/shape follow Slice B's actual `VcWalletClient` API,
   not the design doc §3 sketch.** The sketch (written before Slice B)
   used `issueCredential`/a single issuance entry point and
   `Set<String> consentedClaimPaths` for presentation. Slice B's actual
   package exposes `issuePreAuthorized` plus a separate
   `beginAuthorizationCodeIssuance`/`completeAuthorizationCodeIssuance`
   pair, and `submitPresentation` takes a whole `IssuedCredential` (v1
   always fully re-discloses, no per-claim selection) — the port mirrors
   this exactly, per the Slice C task's explicit instruction to match what
   was actually built. Storage methods also operate on a new
   `StoredVerifiableCredential` type (not a bare `IssuedCredential`) since
   the package deliberately persists nothing itself.
2. **No dedicated screens for the PAR/PKCE/authorization_code fast-follow
   flow.** The design doc's overall plan phases this in as a fast-follow
   "within the same v1 effort," but Slice C's own acceptance criteria (§5)
   only requires the pre-authorized_code round trip end-to-end through the
   UI. `beginAuthorizationCodeIssuance`/`completeAuthorizationCodeIssuance`
   are fully wired on the port (thin pass-throughs to the package) and the
   redirect URI is registered on both platforms, so a future screen can be
   added without touching the port or platform config again — but no UI
   calls them yet. Flagging this as the most likely next follow-up.
3. **No `freezed` for the new sealed state classes**
   (`vc_issuance_flow_state.dart`, `vc_presentation_flow_state.dart`).
   This repo's existing flow-state files (`idv_flow_state.dart`,
   `passkey_flow_state.dart`) use `@freezed`, which requires `build_runner`
   codegen. No Dart/Flutter SDK was available in this environment to run
   `dart run build_runner build` and verify generated `.freezed.dart` output
   (see "Verification status" below), so hand-writing `.freezed.dart` by
   guesswork would have been a real correctness risk with no way to check
   it. Plain Dart 3 sealed classes with object-pattern `switch` give the
   same exhaustive-matching ergonomics used throughout the new screens
   without any generated code. If a working toolchain is available later,
   converting these two files to `@freezed` for consistency is a
   safe, mechanical follow-up (no consumer-facing shape change needed since
   consumers already just do pattern matching).
4. **Added `http: ^1.2.2` as a direct `pubspec.yaml` dependency**, despite
   the task noting "nothing new should be needed." `VcWalletSdkService`'s
   constructor exposes an injectable `httpClient` parameter (forwarded to
   `VcWalletClient.create`) so tests can supply `package:http/testing.dart`'s
   `MockClient` — the same DI pattern Slice B's own package uses internally.
   `http` was already present transitively (via `vc_wallet_client`'s own
   dependency), so this doesn't change what's actually resolved/pinned; it
   only makes an already-used package a declared, direct dependency (the
   correct fix for the `depend_on_referenced_packages` lint rather than
   relying on an undeclared transitive resolution). Version pinned to match
   `packages/vc_wallet_client/pubspec.yaml`'s own constraint exactly, so no
   version-solver conflict is possible. No other line in `pubspec.yaml` was
   touched.
5. **`vc_wallet_routes.dart` lives in `navigation/`, not `presentation/`.**
   The design doc's folder sketch doesn't specify exactly where
   location-builder helpers live; `credential_setup_flow_routes.dart` (the
   named precedent) lives in `credential_setup/navigation/`, so this mirrors
   that placement. (Caught and fixed a real bug during this slice's own
   verification pass: several `presentation/*.dart` screens initially
   imported it as `'vc_wallet_routes.dart'`, a same-directory import that
   would have resolved to the wrong — nonexistent — path; fixed to
   `'../navigation/vc_wallet_routes.dart'` everywhere.)
6. **CI workflow secrets** (`.github/workflows/android-ci.yml`,
   `android-yubico-brand-release.yml`) were **not** updated with
   `SIROSID_VC_*` secret injection, even though design-doc §6 recommends it.
   The Slice C task's explicit concrete-steps list only calls out
   `.env.local.example`; CI wiring is a reasonable, low-risk follow-up but
   was left out to stay strictly within the given scope.

## Verification status

**No Dart/Flutter SDK was available in this environment** (`dart`/`flutter`
both absent from `PATH` and not found on disk) — `flutter pub get`,
`flutter analyze`, and `flutter test` could not actually be run or their
results observed, matching Slice B's own documented limitation.

What was done instead, matching Slice B's compensating rigor:
- Every new/edited file was re-read end-to-end after writing for
  syntax/type/import correctness. This caught and fixed two real bugs:
  - A wrong relative-import path for `vc_wallet_routes.dart` in eight
    `presentation/*.dart` screens plus the dashboard entry-point widget
    (see deviation 5 above) — would have been a compile-time "file not
    found" error in every affected screen.
  - Two `switch` **statements** (in `VcIssuanceFlowController.confirmAndIssue`
    and `VcDeepLinkListener._handleIncomingLink`) whose case bodies ended
    without an explicit `break`/`return` — invalid in Dart outside `switch`
    *expressions* (confirmed against this repo's own existing
    `enrollment_flow_controller.dart` switch statement, which does end every
    case in an explicit jump statement). Fixed by adding `break;`.
- Every dependency's exact API used (`VcWalletClient`, its models,
  `flutter_secure_storage`'s `TestFlutterSecureStoragePlatform` test helper,
  `package:http/testing.dart`'s `MockClient`, Riverpod's
  `AsyncNotifierProvider`/`.future`/`ProviderContainer.listen`) was checked
  against the actual source already present on disk (Slice B's package,
  the app's own existing test files using the same helpers), not from
  memory.
- One test was rewritten after noticing a timing-flakiness risk: an
  earlier draft of the "issuance stores and invalidates wallet-home" test
  asserted on a `container.listen` callback's fire count immediately after
  `await confirmAndIssue()`, which isn't guaranteed to have observed the
  invalidated provider's completed rebuild yet. Rewritten to
  `await container.read(vcWalletHomeControllerProvider.future)` after the
  mutation, which deterministically waits for the rebuild.
- The riskiest untested path was deliberately *not* faked with
  hand-fabricated data: `submitPresentation`'s real (non-fake-port) code
  path always attempts to sign an SD-JWT key-binding JWT via
  `EcP256KeyPair.fromStoredMaterial(holderKey)`, which reconstructs an EC
  point from `HolderKeyMaterial.x`/`y` and would very likely fail
  pointycastle's on-curve validation for arbitrary hand-picked test bytes
  (unlike DPoP/proof-of-possession keys, which the package always
  *generates* fresh and are therefore genuinely valid). The
  `vc_wallet_ports_test.dart` "mocked HTTP" test therefore exercises the
  full `resolveCredentialOffer → issueCredentialPreAuthorized →
  storeIssuedCredential` path (which only ever signs with freshly-generated
  keys) but not `submitPresentation`'s real crypto path; the port's wiring
  for `submitPresentation` (i.e., that it's called correctly with the right
  arguments) is instead covered via the fake-port-backed
  `vc_presentation_flow_controller_test.dart`. Reconstructing a real,
  on-curve EC key pair from scratch without a toolchain to actually run and
  verify it would have been an unverified guess, not a real check — the
  right place for that is Slice B's own test suite, which has access to
  `EcP256KeyPair.generate()` internally.

Given all of the above, results should be treated as **"should pass,
pending a real `flutter pub get && flutter analyze && flutter test` run,"**
not confirmed-green — flagging clearly for whoever next has a working
Flutter toolchain. Recommended first steps: `flutter pub get` (picks up the
new direct `http` dependency; should not change any resolved version),
`flutter analyze`, then `flutter test` (new suite first, then the full
existing suite to confirm the untouched IDV/passkey-manager tests, report
02 §6's ~4,400 lines, remain green).

## Test coverage added

| File | Covers |
|---|---|
| `test/features/verifiable_credentials/domain/vc_deep_link_test.dart` | Scheme/host classification for all three link kinds, case-insensitivity, unrelated/empty/unconfigured-redirect edge cases |
| `test/features/verifiable_credentials/domain/stored_verifiable_credential_test.dart` | `fromIssuedCredential` defaults, `toJson`/`fromJson` round-trip, `toIssuedCredential` re-parse |
| `test/features/verifiable_credentials/domain/consent_claim_test.dart` | Canonical claim ordering/labels for both preview (no values) and populated (with values) cases, unknown-claim fallback |
| `test/features/verifiable_credentials/state/vc_issuance_flow_controller_test.dart` | Offer resolution success/failure/no-config-id, issuance success + wallet-home invalidation, tx-code prompt/retry, terminal error |
| `test/features/verifiable_credentials/state/vc_presentation_flow_controller_test.dart` | Request resolution with/without a stored credential, submit + auto-poll-once-if-submitted, no-double-poll-if-already-accepted, submission failure |
| `test/features/verifiable_credentials/state/vc_wallet_home_controller_test.dart` | Initial load, delete + refresh, single-credential lookup provider |
| `test/app/config/app_sirosid_vc_config_test.dart` | `missingFields()` with nothing configured, `fromEnvironment()` never throwing on empty values |
| `test/app/infrastructure/verifiable_credentials/vc_wallet_ports_test.dart` | Storage round trip + corrupted-payload handling (via `flutter_secure_storage`'s real `TestFlutterSecureStoragePlatform`), and a real (non-faked) `resolveCredentialOffer → issueCredentialPreAuthorized → storeIssuedCredential` round trip against a mocked HTTP client, mirroring Slice B's own test style |
| `test/features/verifiable_credentials/presentation/vc_wallet_home_screen_test.dart` | Empty state, populated list + navigation to detail, "Add credential"/"Present a credential" navigation |
| `test/features/verifiable_credentials/presentation/vc_issuance_consent_screen_test.dart` | Claims-preview rendering once an offer resolves; accept → progress → detail screen navigation, end to end through real routes |

Camera-driven screens (`VcOfferEntryScreen`, `VcPresentationEntryScreen`)
have no dedicated tests, matching this codebase's existing convention —
`EnrollmentQrCodeScreen` (the closest precedent, also `mobile_scanner`-based)
has no test file either.
