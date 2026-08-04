# Hippo Yubico App User — Enrollment / IDV / Credential-Setup Flow Analysis

Analysis target: `c:\Users\Brent.Zundel\dev\yubico\hippo-yubico-app-user`
Scope: app-side Flutter code only (feature folders, ports, config, tests). SDK package
internals (`yubico_sdk_flutter`) and the sirosid backend are covered by other reports.
This document feeds a later design-synthesis phase for swapping/augmenting the
"YubicoId" digital-identity-credential backend with sirosid's OpenID4VCI/OpenID4VP stack.

---

## 0. TL;DR for the design phase

- There is exactly one seam in the app that is IDV-backend-aware at the protocol
  level: `YubicoIdSdkPort` (3 methods) plus the `idvProvider` string returned inside
  its responses, which the app already branches on today (`'privateid'` vs. anything
  else = "BYO IDV", a generic external-browser-redirect + deep-link-callback path).
  That `idvProvider` branch is the most natural place to plug in a sirosid-hosted flow
  **for the IDV step**. It says nothing about credential issuance/presentation.
- Credential **minting** (`createSoftwarePasskey` / `createHardwarePasskey` — pure
  FIDO2, no backend awareness) is completely separate from credential **submission**
  (`submitCredential` for the generic enrollment backend vs. `provideYubicoIdCredentials`
  for the YubicoId backend). Only the submission step is YubicoId/vendor aware.
- Nothing in the app today models OpenID4VCI/OpenID4VP concepts (credential offers,
  issuer metadata, presentation exchange, claims/consent). The existing
  `YubicoIdSdkPort`/`YubicoIdResponse.state` state machine is a hand-rolled,
  stringly-typed protocol (`'idv_required'`, `'passkey_creation_required'`,
  `'software'`, `'hardware'`, `'success'`, `'failed'`) that looks purpose-built for a
  different (Yubico-hosted) backend, not sirosid. A real OpenID4VCI/VP integration
  will likely need new UI states and probably new port methods rather than reusing
  this response contract verbatim.
- Config plumbing is 100% compile-time `--dart-define` values, sourced from a
  gitignored `.env.local` file (VS Code launch configs + terminal) and from GitHub
  Actions secrets baked into a temp dart-defines file at CI/build time. A sirosid
  config surface (wallet-backend base URL, environment name, client id, etc.) would
  follow the exact same pattern.
- Test coverage for these flows is substantial (enrollment controller: 735 lines,
  IDV controller: 1821 lines, credential-setup navigation: 1243 lines, plus deep-link
  and domain-parsing tests) and should be extended, not replaced, by whoever
  implements the sirosid integration.

---

## 1. End-to-end flow trace

### 1.1 Entry: QR code / one-time code / remote link → enrollment token

- `lib/features/enrollment/presentation/enrollment_qr_code_screen.dart` scans a QR
  code and calls `EnrollmentFlowController.onQrCodeScanned` (controller:
  `lib/features/enrollment/state/enrollment_flow_controller.dart:97-137`), which calls
  `EnrollmentSdkPort.parseOnboardingToken(rawQrCode)` at line 114-116.
- Alternatively, a universal/custom-scheme link is handled by
  `lib/app/navigation/remote_enrollment_link_listener.dart:51-90`, which parses it
  with the domain type `RemoteEnrollmentLink.parse` (see §2) and routes to
  `EnrollmentFlowController.onRemoteEnrollmentLinkReceived`
  (`enrollment_flow_controller.dart:139-145`), which shares the same
  `_acceptParsedEnrollmentToken` path as the QR flow (`enrollment_flow_controller.dart:190-200`).
- Both entry points converge on `_startYubicoIdFlow` (`enrollment_flow_controller.dart:260-310`):
  1. `_ensureSdkEnvironment(environment)` → `SdkLifecyclePort.ensureInitialized(environment:)`
     (`enrollment_flow_controller.dart:272`, helper at `713-731`).
  2. `yubicoIdDeviceIdStoreProvider.getOrCreate()` — a locally-generated/persisted
     device id (`enrollment_flow_controller.dart:277-279`).
  3. **`YubicoIdSdkPort.startYubicoIdFlow(token, deviceId)`** —
     `enrollment_flow_controller.dart:284-286`. This is the first vendor-backend call
     in the whole flow.
  4. Result is a `YubicoIdResponse` dispatched by `_handleYubicoIdResponse`
     (`enrollment_flow_controller.dart:353-413`) on `response.state`:
     - `'idv_required'` / `'idv'` → `_handleYubicoIdIdvRequired` (`415-502`)
     - `'passkey_creation_required'` or `'software'`/`'hardware'` →
       `_handleYubicoIdCredentialRequired` (`525-580`)
     - `'success'` → `EnrollmentUiState.complete()`
     - `'failed'` / unknown → error state

  Note: there is also a **legacy/manual path** that never touches `YubicoIdSdkPort`:
  `submitOneTimeCode` → `_fetchCreateOptions` (`enrollment_flow_controller.dart:619-692`)
  calls **`EnrollmentSdkPort.fetchCreateOptions(token, oneTimeCode)`** directly and
  goes straight to `EnrollmentUiState.readyToCreate`. This is the generic (non-YubicoId)
  enrollment backend, used when `FEATURE_ONE_TIME_CODE_FLOW_ENABLED` is on (see §5).
  It is a structurally separate integration point from the YubicoId flow and probably
  out of scope for the sirosid swap, but a future design should confirm this.

### 1.2 IDV branch: Private ID vs. "BYO IDV"

`_handleYubicoIdIdvRequired` (`enrollment_flow_controller.dart:415-502`) reads
`idvProvider = data?.idvProvider ?? 'privateid'` and calls
`_isPrivateIdProvider(idvProvider)` (`761-768`, normalizes case/separators and treats
empty or `'privateid'` as the native provider — **everything else is "BYO IDV"**).

- **Private ID branch** (native, in-app camera capture):
  - Persists a pending credential request if `createOptions`/`credentialUuid` are
    present (`_persistYubicoIdCredentialRequestIfPresent`, `504-523`).
  - Depending on local onboarding-progress milestones
    (`onboardingProgressStoreProvider.hasCompletedInitialCredentialSetup` /
    `.hasCompletedIdvWithMockNfc()`), routes into one of:
    - `IdvFlowController.reviewYubicoIdIdvSession(token, idvData, credentialUuid, creationOptions)`
      (`idv_flow_controller.dart:128-161`) — first-time IDV, shows the document/country
      picker review screen.
    - `IdvFlowController.startYubicoIdReauthSession(...)` (`idv_flow_controller.dart:564-589`)
      — face-only re-auth if mock-NFC milestone already passed.
    - `IdvFlowController.startYubicoIdIdvSession(token, idvData, credentialUuid, creationOptions)`
      (`idv_flow_controller.dart:505-518`) — starts the full IDV session immediately.
  - All three eventually call the private `_startYubicoIdSession` helper
    (`idv_flow_controller.dart:591-690`), which builds an `IdvSessionConfig` via
    `_yubicoIdIdvSessionConfig(idvData)` (`538-562`) — this **merges the backend's
    returned `idvData` map (session_id/user_id/api_key/base_url/product_group_id/
    redirect_url/callback_url) over the app's `--dart-define` `PRIVATE_ID_*` defaults**
    (`AppIdvSessionConfig.fromEnvironment`, see §5) — and calls
    **`IdentificationSdkPort.startIdvSession(config)`** (`idv_flow_controller.dart:658`).
  - From there the flow is driven entirely by `IdentificationSdkPort`: camera
    permission (`getCameraPermissionStatus`/`requestCameraPermission`), the
    `idvStateChanges` stream (subscribed once in `build()`,
    `idv_flow_controller.dart:64-71`), and explicit step calls —
    `startIdvLiveness` (930), `retryIdvLiveness` (799-803), `startIdvDocumentScan`
    (943), `retryIdvDocumentScan` (805-809), `confirmIdvDocumentPreview` (956),
    `startIdvNfcScan` (966), `retryIdvNfc` (811-815), `stopIdvNfcScan` (979),
    `triggerIdvVerification` (989-1016), `cancelIdvSession` (via
    `_cancelIdvSessionSilently`, 1338-1344). Screen-to-phase routing lives in
    `lib/features/idv/presentation/idv_flow_route_resolver.dart:6-42`
    (`IdvPhase` → route, e.g. `readyForLiveness` → liveness intro,
    `documentScan` → capture screen, `nfcWait` → NFC screen, `complete` →
    `idvYubicoIdFinalizingScreen` if `idvFlow.isYubicoIdContinuation` else
    `idvResultScreen`).
  - This whole `IdentificationSdkPort` state machine (`IdvState`/`IdvPhase`/
    `IdvSessionConfig` with fields like `documentGuidance`, `faceGuidance`,
    `mugshotBytes`, `isNfcScanning`) is **Private-ID-flavored plumbing** reused
    for both the plain IDV flow and the YubicoId-continuation IDV flow. If sirosid
    keeps using Private ID for the actual capture/liveness/NFC work and only
    replaces what happens *after* IDV (credential issuance), this layer is
    untouched. If sirosid supplies its own IDV, this camera-driven state machine
    is irrelevant to the swap and the BYO-IDV path below is the more relevant seam.

- **"BYO IDV" branch** (`_startByoIdvFlow`, `enrollment_flow_controller.dart:582-617`):
  - Expects `idvData['url']` to be a valid absolute URL.
  - Sets `EnrollmentUiState.awaitingByoIdvCallback(token, environment, idvProvider, idvUrl)`.
  - Opens it with `externalUrlLauncherProvider(uri)` (system browser/webview launch).
  - The app then waits for a **custom-scheme deep link** parsed by
    `ByoIdvCallbackLink.tryParse` (see §2) in
    `lib/app/navigation/remote_enrollment_link_listener.dart:57-63`, which calls
    `EnrollmentFlowController.handleByoIdvCallback(rawCallbackUrl)`
    (`enrollment_flow_controller.dart:230-245`) → `_provideYubicoIdv(token, environment,
    idvData: {'callback_url': trimmedCallbackUrl}, bypassCreatePasskeyReauth: true)`
    (`312-351`) → **`YubicoIdSdkPort.provideYubicoIdv(token, idvData)`**
    (`325-327`) → re-dispatches through `_handleYubicoIdResponse` again.
  - **This is the pluggable-IDV seam most relevant to sirosid**: any IDV vendor
    that can (a) be launched via URL and (b) complete by redirecting back to a
    custom-scheme deep link carrying enough info for the backend to look up the
    result, can be wired in purely by having the backend return a non-`privateid`
    `idvProvider` + `url`. No app code changes would be needed for the *browser
    hand-off* itself — only for what the app does with the callback data (currently
    it just forwards `callback_url` back to `provideYubicoIdv`, trusting the backend
    to resolve the session).

### 1.3 Post-IDV → credential-required response → credential type selection

Whether IDV completed via Private ID or BYO, the response is funneled back through
`_handleYubicoIdCredentialRequired` (`enrollment_flow_controller.dart:525-580`) or,
after camera-based IDV succeeds, through
`lib/features/credential_setup/navigation/credential_setup_navigation.dart` :

- `IdvYubicoIdFinalizingScreen` (`lib/features/idv/presentation/idv_yubico_id_finalizing_screen.dart:38-45`)
  calls `continueAfterSuccessfulIdv(container)` on its first frame, then
  `context.go(nextLocation)`.
- `continueAfterSuccessfulIdv` (`credential_setup_navigation.dart:74-79`) calls
  `IdvFlowController.completeSuccessfulIdv()` then `_continueAfterSuccessfulIdv`
  (`231-244`) → `_continueYubicoIdAfterIdv` (`246-280`), which calls
  **`EnrollmentFlowController.continueAfterSuccessfulYubicoIdIdv(token, environment,
  idvData)`** (`enrollment_flow_controller.dart:247-258`) → internally
  **`YubicoIdSdkPort.provideYubicoIdv`** again — i.e., the *same* method is used both
  to hand IDV results to the backend and to report BYO-IDV callback results; the
  backend's response state machine decides what happens next.
- If the response now carries `createOptions`, routing goes to
  `postYubicoIdvRouteForResolvedCredentialAsync` (`credential_setup_navigation.dart:282-329`),
  which seeds either:
  - `PasskeyFlowController.seedSetupDetails(...)` → route
    `AppRoutes.softwarePasskeyDetailsScreen`, or
  - `HardwareKeyFlowController.seed(...)` → route `AppRoutes.hardwareKeyPlaceholder`
  based on `CredentialRegistrationType` (`software`/`hardware` derived from the
  YubicoId response state via
  `credentialRegistrationTypeFromYubicoIdState` in
  `lib/app/providers/selected_credential_registration_type_provider.dart:3-10`).
- The **credential type is decided by the backend response**, not by the
  `CredentialPickerScreen` UI (`lib/features/credential_setup/presentation/credential_picker_screen.dart`)
  in the YubicoId path — that screen is only reached in the generic/manual
  (`fetchCreateOptions`) enrollment path via `continueWithResolvedCredential` →
  `_routeForRequiredIdv` → ... → `postIdvRouteForResolvedCredentialAsync`
  (`credential_setup_navigation.dart:37-46, 380-421`).

### 1.4 Credential creation + submission (FIDO2 minting vs. backend submission)

- **Software passkey** (`lib/features/software_passkey/state/passkey_flow_controller.dart`):
  - `createPasskey()` (`100-184`) calls **`SoftwarePasskeySdkPort.createSoftwarePasskey(options, reauthProvider)`**
    (`123-127`) — pure FIDO2, backend-agnostic.
  - Then `_submitCredential(details, credential)` (`208-228`) branches:
    - if `details.submissionContext is YubicoIdCredentialSubmissionContext` →
      **`YubicoIdSdkPort.provideYubicoIdCredentials(token, credentialUuid, credential)`**
      (`220-226`);
    - else → **`SoftwarePasskeySdkPort.submitCredential(token, credential)`**
      (`215-217`, generic enrollment backend).
  - Response mapped to `EnrollmentResult` via `_enrollmentResultFromYubicoIdResponse`
    (`271-285`), same stringly-typed `'success'`/`'failed'`/other convention as the
    enrollment controller.

- **Hardware key** (`lib/features/hardware_key/state/hardware_key_creation_controller.dart`):
  - `_createHardwareKeyFromDetails` (`93-188`) calls
    **`HardwareKeySdkPort.createHardwarePasskey(options, pinProvider, ...)`**
    (`115-125`) — again pure FIDO2/CTAP, backend-agnostic, with PIN and NFC-retap
    callbacks.
  - `_submitCredential` (`427-447`) has the identical branch: YubicoId
    (`provideYubicoIdCredentials`, `439-445`) vs. generic
    (`HardwareKeySdkPort.submitCredential`, `434-436`).
  - After successful submission, records the hardware key locally
    (`recordEnrolledHardwareKey`, `HardwareKeySdkPort`) — this is passkey-manager
    bookkeeping, not IDV/credential-issuance concern.

### 1.5 Finalization

- `IdvYubicoIdFinalizingScreen` (`lib/features/idv/presentation/idv_yubico_id_finalizing_screen.dart`)
  is a pure "await async continuation, then navigate" screen — no direct SDK calls
  itself; all work happens in `continueAfterSuccessfulIdv`. This is the natural
  screen to instrument/replace if sirosid's post-IDV continuation needs a different
  async shape (e.g., polling an OpenID4VCI credential-offer/issuance status) — it
  already tolerates arbitrary await time with a generic loading state
  (`idv_yubico_id_finalizing_screen.dart:47-67`).
- `IdvFinalizingScreen` (`lib/features/idv/presentation/idv_finalizing_screen.dart`) is
  the *non*-YubicoId counterpart (`IdvPhase.finalising`, reached for standard IDV
  without a YubicoId continuation) — cosmetically identical but functionally inert
  (just a loading screen keyed to route `AppRoutes.idvFinalizingScreen`; the actual
  polling happens inside the SDK's IDV session, not here).
- `IdvResultScreen` (`lib/features/idv/presentation/idv_result_screen.dart`) is shown
  for the generic (non-YubicoId) IDV success case and for face-only verify/reauth
  sessions; its `onContinue` callback is wired by the router to
  `postIdvRouteForResolvedCredentialAsync` and friends.

---

## 2. `byo_idv_callback_link.dart` and `remote_enrollment_link.dart`

Both are self-contained URI-parsing value types with no SDK/network dependencies —
pure domain logic, easy to test in isolation and easy to extend.

- **`RemoteEnrollmentLink`** (`lib/features/enrollment/domain/remote_enrollment_link.dart`):
  Parses either the custom scheme `yubicosoftkeys://credentials?token=...&environment=...`
  or the Android App Link `https://enrollment.yedaudit.yubico.com/credentials?...`
  into a `{token, environment}` pair. `environment` is the **same enum-like string**
  (`demo`/`dev`/`stage`/`prod`) used to pick the SDK's backend environment
  (`_ensureSdkEnvironment`, `enrollment_flow_controller.dart:713-731`, and
  `_sdkEnvironmentFromName` in `yubico_sdk_service.dart:83-95`). This confirms the
  app already supports **multiple named backend environments end-to-end** — this is
  a strong, ready-made hook for a sirosid deployment axis (e.g. adding a `sirosid`/
  `siros-staging` environment value), though today the set of valid values is
  hardcoded to `{demo, dev, stage, prod}` in `enrollment_flow_controller.dart:720`.

- **`ByoIdvCallbackLink`** (`lib/features/enrollment/domain/byo_idv_callback_link.dart`):
  Matches links with host `idv` and path `/callback` on **any non-http(s) custom
  scheme** (`uri.scheme.isEmpty || uri.scheme == 'http' || uri.scheme == 'https'` is
  explicitly rejected, `byo_idv_callback_link.dart:18-20` — i.e. it deliberately
  accepts arbitrary custom app-link schemes, not just one hardcoded scheme name).
  This is intentionally generic; it is the return address for *any* external IDV
  provider redirect, matching the `_startByoIdvFlow`/`handleByoIdvCallback` pair
  described in §1.2.

**Do these suggest a pluggable/configurable IDV backend?** Yes, but only for the
*identity verification step*, and the pluggability lives on the **backend response**
side (`idvProvider` field + `url`), not in any app-side vendor registry or config
file. The app itself has no notion of "which IDV vendor is configured for this
deployment" beyond the hardcoded `_isPrivateIdProvider` string check
(`enrollment_flow_controller.dart:761-768`) — it is reactive, not configurative: it
does whatever the backend's `idvProvider` value tells it to do, either driving the
in-app camera SDK (Private ID) or bouncing to a browser and waiting for a deep link
(anything else). There is nothing here that already anticipates OpenID4VCI/VP
credential issuance or presentation — the pluggability is scoped strictly to IDV,
not to the credential lifecycle that follows it.

---

## 3. Boundary: "digital identity credential" vs. "FIDO2 passkey manager"

**Identity-credential concern (YubicoId enrollment/IDV/issuance) — touch this for sirosid:**
- Ports: `YubicoIdSdkPort` (all 3 methods), `EnrollmentSdkPort` (both methods —
  `parseOnboardingToken`, `fetchCreateOptions`), `IdentificationSdkPort` (all of it —
  IDV session lifecycle), and the `submitCredential`/`provideYubicoIdCredentials`
  branch inside `SoftwarePasskeySdkPort`/`HardwareKeySdkPort` (submission only, not
  creation).
- Features: `lib/features/enrollment/**`, `lib/features/idv/**`,
  `lib/features/credential_setup/**` (navigation/routing + the two "credential type"
  screens), plus the `CredentialSubmissionContext`/`YubicoIdCredentialSubmissionContext`
  domain type in
  `lib/features/credential_setup/domain/credential_creation_display_data.dart:54-68`,
  which is the piece of state that threads "this credential submission must go to
  YubicoId, not the generic enrollment backend" through the passkey/hardware-key
  controllers.
- Dashboard-side: `lib/features/dashboard/state/pending_credentials_controller.dart`
  and `lib/features/dashboard/domain/pending_credential_summary.dart` surface
  **in-progress enrollment requests** (from `PendingCredentialRequestStore`,
  `lib/app/infrastructure/stores/pending_credential_request_store.dart`) on the
  dashboard home screen — this is enrollment-adjacent, not passkey-manager.

**FIDO2 passkey-manager concern — separate, do not conflate, per the existing facts:**
- Ports: `CredentialsSdkPort` (list/assert/update-metadata/delete for arbitrary
  website passkeys), `PasskeyProviderPort` (OS credential-provider settings), and
  the *creation* halves of `SoftwarePasskeySdkPort.createSoftwarePasskey` /
  `HardwareKeySdkPort.createHardwarePasskey`/PIN/reset/list-enrolled-keys methods —
  these are pure WebAuthn/CTAP operations with no backend awareness at all.
- Features: `lib/features/credentials/**` (list/details/favicon/assertion for
  arbitrary relying parties) and `lib/features/yubikeys/**` (enrolled hardware key
  management/rename/delete), both driven from `lib/features/dashboard/**`'s "your
  passkeys" surfaces — confirmed via `pending_credentials_controller.dart` living
  alongside but structurally separate from `yubikeys_controller.dart`/
  `credentials_controller.dart`.

**The dividing line in code, precisely:** `createSoftwarePasskey`/`createHardwarePasskey`
mint a WebAuthn credential regardless of *why* — same call whether the flow is
website-passkey creation or YubicoId enrollment. What changes per-concern is only the
**submission** call directly after minting (`submitCredential` vs.
`provideYubicoIdCredentials`), gated by `CredentialSubmissionContext`. A sirosid
design should not need to touch credential *minting* at all — only the submission
branch, the `YubicoIdSdkPort` methods, `EnrollmentSdkPort`, `IdentificationSdkPort`,
and the enrollment/idv/credential_setup navigation glue.

---

## 4. Screens/controllers most likely to need changes for a sirosid OpenID4VCI/OpenID4VP backend

**Likely need new UI states / meaningfully new logic:**
- `lib/features/idv/presentation/idv_yubico_id_finalizing_screen.dart` +
  `continueAfterSuccessfulIdv`/`_continueYubicoIdAfterIdv` in
  `credential_setup_navigation.dart:74-280` — this is the natural home for an
  OpenID4VCI credential-offer/issuance polling step; today it's a single opaque
  await with no intermediate states (no "issuer selected", "credential type chosen",
  "waiting for issuance", "claims preview/consent").
- `enrollment_flow_controller.dart`'s `_handleYubicoIdResponse` switch
  (`353-413`) and the `YubicoIdResponse.state` string enum — if sirosid's protocol
  doesn't map cleanly onto `idv_required`/`passkey_creation_required`/`software`/
  `hardware`/`success`/`failed`, this whole dispatch needs new states, e.g. for
  issuer/credential-type selection (OpenID4VCI credential offers can present
  multiple `credential_configuration_ids`) or presentation/consent
  (OpenID4VP would need a "what are you sharing, to whom, do you consent" screen —
  there is **no consent/claims-preview screen anywhere in this codebase today**).
- `CredentialPickerScreen` (`lib/features/credential_setup/presentation/credential_picker_screen.dart`)
  currently offers exactly two hardcoded options ("Mobile passkey" / "YubiKey"); if
  sirosid needs a credential-*type* or issuer picker (distinct from *authenticator*
  type), this screen's pattern (simple radio-card list + continue button) could be
  reused but the underlying `CredentialRegistrationType` enum
  (`lib/app/providers/selected_credential_registration_type_provider.dart:1`,
  currently only `softwarePasskey`/`hardwareKey`) is too narrow — it conflates
  "authenticator form factor" with what would need to become "credential type from
  issuer," so it likely needs to stay as-is and a *new* concept added alongside it
  rather than repurposing it.
- `YubicoIdSdkPort` itself: 3 methods (`startYubicoIdFlow`, `provideYubicoIdv`,
  `provideYubicoIdCredentials`) with no notion of issuer metadata, credential
  offers, scopes/claims, or presentation requests. A sirosid integration will very
  likely need new port methods here (or a parallel port) rather than reusing these
  three verbatim, since OpenID4VCI/VP have request/response shapes (credential
  offer URIs, token endpoints, credential endpoints, presentation definitions) that
  don't fit the current `Map<String, String> idvData` / opaque `CreatedCredential`
  parameters.

**Likely unaffected / reusable as-is:**
- All of `IdentificationSdkPort` and the camera/liveness/NFC/document-scan screens
  under `lib/features/idv/presentation/` (everything except the two "finalizing"
  screens) — these only matter if sirosid also replaces the *IDV* vendor, not just
  the *credential issuance* backend. If Private ID remains the IDV provider and only
  post-IDV issuance moves to sirosid, none of this needs to change.
- `SoftwarePasskeySdkPort.createSoftwarePasskey` / `HardwareKeySdkPort.createHardwarePasskey`
  and the hardware key PIN/NFC-retap ceremony screens
  (`lib/features/hardware_key/presentation/**`) — pure FIDO2 minting, backend-agnostic.
- `CredentialsSdkPort`, `PasskeyProviderPort`, and all of
  `lib/features/credentials/**` / `lib/features/yubikeys/**` / dashboard passkey
  surfaces — genuinely separate concern, as established.
- `EnrollmentFlowController`'s QR/one-time-code parsing and
  `RemoteEnrollmentLink`/`ByoIdvCallbackLink` domain parsing — protocol-agnostic
  entry mechanics; sirosid would reuse the same QR/link/one-time-code *transport*,
  just producing a different backend response downstream.
- `HardwareKeyFlowController`/`PasskeyFlowController`'s *creation* logic — only their
  `_submitCredential` branch is YubicoId-aware; the rest (PIN handling, touch/NFC
  callbacks, reauth-provider wiring) is unaffected.

---

## 5. Config/build plumbing

All backend-specific configuration is **compile-time only**, via `--dart-define`
(`String.fromEnvironment`/`bool.fromEnvironment`/`int.fromEnvironment`). There is no
runtime config-fetch mechanism for these values (contrast with
`remoteFeatureFlagsProvider` in `feature_flags.dart:107-109`, which is a stub
returning an always-empty `FeatureFlags` — a placeholder for a *future* remote flag
source, not yet implemented).

- **Definition site**: `lib/app/config/app_idv_session_config.dart:4-25` —
  `AppIdvSessionConfig.fromEnvironment()` reads `PRIVATE_ID_API_KEY`,
  `PRIVATE_ID_BASE_URL`, `PRIVATE_ID_PRODUCT_GROUP_ID`, `PRIVATE_ID_REDIRECT_URL`,
  `PRIVATE_ID_CALLBACK_URL`, `PRIVATE_ID_USER_ID` into an SDK `IdvSessionConfig`, and
  `AppIdvSessionConfig.missingFields()` is used to fail fast with a clear message
  ("Missing IDV configuration. Set these --dart-define values: ...") in
  `idv_flow_controller.dart` (e.g. lines 270-296, 432-455, 603-632).
- **Local dev file**: root `.env.local.example` lists the six `PRIVATE_ID_*` keys
  (plus unrelated `CLIENT_ID`/`CLIENT_SECRET`/etc. used only by
  `scripts/generate_credential.sh`, not by the app itself). Developers copy it to
  `.env.local` (gitignored) per `README.md:16-38`.
  - **`otherBrand` flavor**: The `.env.local` file is shared across brands/flavors —
    there's no per-flavor override; `otherBrand` VS Code configs load the same
    `.env.local` as `yubico`. Whether sirosid config should also be single-file or
    per-brand is an open question for the design phase.
- **VS Code**: `.vscode/launch.json` — every configuration passes
  `--dart-define-from-file=.env.local` plus `--dart-define=BRAND=<flavor>` and
  `--dart-define=IOS_APP_GROUP_IDENTIFIER=...`; six configurations total (3 build
  modes × 2 brands).
- **CI**: `.github/workflows/android-ci.yml:59-79` writes GitHub Actions secrets
  (`secrets.PRIVATE_ID_*`) into a temp file `$RUNNER_TEMP/app_dart_defines.env` and
  passes it via `--dart-define-from-file` to both `flutter test` and
  `flutter build appbundle`; missing secrets only produce a `::warning::` (CI still
  runs with empty IDV dart-defines). `.github/workflows/android-yubico-brand-release.yml`
  follows the identical pattern but **hard-fails** on missing secrets
  (`: "${PRIVATE_ID_API_KEY:?Missing GitHub secret PRIVATE_ID_API_KEY}"`, lines
  63-68) before building the release artifact.
- **A sirosid config surface would follow this exact pattern**: add
  `SIROSID_*` (e.g. `SIROSID_BASE_URL`, `SIROSID_ENVIRONMENT`/reuse the existing
  `environment` query param already threaded through
  `RemoteEnrollmentLink`/`_ensureSdkEnvironment`, `SIROSID_CLIENT_ID`, etc.) to (1)
  `.env.local.example`, (2) a new `AppSirosIdConfig.fromEnvironment()` analogous to
  `AppIdvSessionConfig`, (3) both GitHub Actions workflows' secret-injection blocks,
  and (4) nothing needed in `.vscode/launch.json` since it already forwards the
  whole `.env.local` file. Note the existing hardcoded environment allow-list
  (`{'demo','dev','stage','prod'}`, `enrollment_flow_controller.dart:720`) would need
  to either accept new values or be replaced with a more general mechanism if sirosid
  introduces its own environment naming.

---

## 6. Existing test coverage to be aware of / extend

Large, relevant existing suites (line counts to gauge depth):

| Test file | Lines | Covers |
|---|---|---|
| `test/features/idv/state/idv_flow_controller_test.dart` | 1821 | Full IDV phase state machine, session config merging (incl. YubicoId idvData overrides), reauth sessions, mock-NFC, failure injection |
| `test/features/credential_setup/navigation/credential_setup_navigation_test.dart` | 1243 | All routing decisions post-IDV / post-YubicoId-IDV, credential-type resolution, skipped-IDV recovery paths |
| `test/features/enrollment/state/enrollment_flow_controller_test.dart` | 735 | QR/link parsing, `_startYubicoIdFlow`, BYO-IDV callback handling, `YubicoIdResponse.state` dispatch, pending-credential persistence |
| `test/features/software_passkey/state/passkey_flow_controller_test.dart` | 643 | Passkey creation + both submission branches (generic vs. YubicoId) |
| `test/app/navigation/remote_enrollment_link_listener_test.dart` | 201 | Deep-link dispatch: BYO-IDV callback vs. remote enrollment link vs. resume-target routing |
| `test/features/idv/presentation/idv_flow_route_resolver_test.dart` | 91 | `IdvPhase` → route mapping, incl. YubicoId-continuation branch to `idvYubicoIdFinalizingScreen` |
| `test/features/enrollment/domain/remote_enrollment_link_test.dart` | 84 | Custom-scheme + Android App Link parsing |
| `test/features/credential_setup/navigation/credential_setup_flow_routes_test.dart` | 70 | Route-path builder functions |
| `test/features/enrollment/domain/byo_idv_callback_link_test.dart` | 35 | Scheme/host/path matching rules |

Also relevant but not read in depth: `test/app/infrastructure/yubico_sdk/yubico_sdk_service_test.dart`
(pass-through service wiring), `test/app/infrastructure/stores/yubico_id_device_id_store_test.dart`,
`test/app/infrastructure/stores/secure_pending_credential_request_store_test.dart`.

**Implication for implementers**: any change to `YubicoIdSdkPort`, the
`YubicoIdResponse.state` dispatch, or the post-IDV navigation functions in
`credential_setup_navigation.dart` will require updates across at least the first
four rows above (~4,400 lines of existing tests) — budget for this. The tests
appear to use a fake/mock SDK layer (see `test/support/fake_yubico_sdk_data.dart`
and `test/support/navigation_test_harness.dart`) which should be the extension point
for sirosid-shaped fakes rather than hitting real network code.

---

## 7. Ambiguous / dead / placeholder code flagged during this analysis

- **`lib/features/credential_setup/presentation/setup_link_screen.dart`** — the
  screen reachable via "Open setup link from your email" on
  `SetupEntryScreen` (`lib/features/credential_setup/presentation/setup_entry_screen.dart:84-88`,
  route `credentialSetupManualFlowLocation()`/`AppRoutes.identificationManualVerify`)
  is a literal placeholder: title "Manual Verification", body text "Manual
  verification will go here.", and a button labeled `"text"` that only triggers a
  biometric-auth check before continuing (`setup_link_screen.dart:28-62`). **Not a
  real implementation** — do not treat this as an existing manual-entry integration
  point; it will need to be built from scratch if sirosid needs a manual/email-link
  path distinct from the QR/remote-link flow already analyzed in §1.1.
- **`lib/features/enrollment/domain/yubico_id_idv_data.dart`** — despite living
  alongside `remote_enrollment_link.dart`/`byo_idv_callback_link.dart` and being
  named like a domain model, this file contains exactly one line: a constant
  `skippedYubicoIdIdvData = {'skipped': 'skipped'}` used only to mark a
  mock-NFC/dev-skip continuation (`enrollment_flow_controller.dart:463`,
  `credential_setup_navigation.dart:90`). It is **not** a rich IDV-data domain type;
  treat it as a magic sentinel value, not an extension point.
- **`_isPrivateIdProvider`** (`enrollment_flow_controller.dart:761-768`) hardcodes
  `'privateid'` as the one recognized native provider string; every other value
  (including empty-after-normalization edge cases already treated as `privateid`)
  falls through to the generic BYO-IDV browser-redirect path. There is no
  app-side allow-list or enum of known providers — the app is purely reactive to
  whatever string the backend sends. A sirosid design should decide whether
  sirosid's IDV (if any) is meant to present itself as a new `idvProvider` value
  (naturally falling into the existing BYO-IDV path) or whether IDV stays on
  Private ID entirely and only the post-IDV credential step changes.
- **`_placeholderYubicoIdCreateOptions()`** (`idv_flow_controller.dart:1486-1506`) —
  a synthetic, non-functional `CredentialCreationOptions` (empty challenge bytes,
  fixed dummy relying-party/user) used only to satisfy `IdvFlowDetails` display
  fields when `reviewYubicoIdIdvSession` is invoked without real `creationOptions`
  yet available. It is cosmetic/display-only, not a fallback credential — do not
  mistake it for a working default.
- **`remoteFeatureFlagsProvider`** (`lib/app/config/feature_flags.dart:107-109`) is a
  stub `Provider` always returning `const FeatureFlags({})` — infrastructure for a
  future remote-config source that doesn't exist yet. Not YubicoId/sirosid-specific,
  but relevant context if the design phase considers a runtime (non-dart-define)
  config channel for sirosid settings — no such channel currently exists anywhere
  in the app.
- **One-time-code / `fetchCreateOptions` path** (`enrollment_flow_controller.dart:619-692`,
  gated by `FEATURE_ONE_TIME_CODE_FLOW_ENABLED`) is a structurally distinct, simpler
  "generic enrollment" flow that never touches `YubicoIdSdkPort` at all. It's unclear
  from app code alone whether this is still an active product path or a legacy/
  parallel one being phased out in favor of the YubicoId flow — flagging as
  ambiguous scope for the design phase to confirm with product/backend owners.
