# yubico_sdk_flutter — Internal Analysis

Repo analyzed: `c:\Users\Brent.Zundel\dev\yubico\hippo-yubico-sdk-flutter` (private Flutter plugin, origin `git@github.com:yubico/hippo-yubico-sdk-flutter.git`)

Scope: this SDK package's internals only — how it is built, what it wraps, and whether it can be extended in-repo to add OpenID4VCI/OpenID4VP and/or R2PS-style remote key provisioning. The consuming app's UI/state machine and the sirosid backend are covered by companion reports.

---

## 0. TL;DR verdict (see §3 for full reasoning)

**(b) Partially extensible — with the "extensible" part being much larger than it first appears, and the "opaque" part being narrower than the method names suggest.**

- The Dart layer (`lib/`), the Pigeon schema (`pigeons/yubico_api.dart`), and the **entire native plugin glue layer** (`android/src/main/kotlin/.../YubicoSdkFlutterPlugin.kt`, `ios/.../YubicoSdkFlutterPlugin.swift`, plus the IDV/camera/enrollment helper classes around them) are ordinary source files in this repo, fully editable, and already wired with production-grade HTTP/crypto/serialization dependencies (Retrofit, OkHttp, kotlinx.serialization, BouncyCastle on Android; OpenSSL.xcframework on iOS).
- However, **every one of the methods named in the mandate — `startYubicoIdFlow`, `provideYubicoIdv`, `provideYubicoIdCredentials`, the `startIdv*`/IDV session-machine internals, `createSoftwarePasskey`/`createHardwarePasskey`/`listHardwareCredentials`/`assertCredential`, `parseOnboardingToken`/`fetchCreateOptions`** — is implemented by a **one-line delegation** from the plugin glue into a closed-source native SDK object (`com.yubico.sdk.*` on Android / `YubicoSDK`/`YubicoSDKCore`/`YubicoSDKEnrollment`/`YubicoSDKIDV`/`YubicoSDKPasskeys` on iOS) that ships into this repo **only as compiled binaries** (`.aar` classes.jar + `.so`, or `.xcframework`). The actual OAuth/HTTP calls to Yubico's backend, the IDV vendor integration, and the passkey/WebAuthn logic live in a **separate, not-checked-out** repo (`Yubico/hippo-yubico-wallet-sdk-android` and `Yubico/hippo-yubico-wallet-sdk-ios`) and are opaque from here.
- Practical implication: **you cannot modify or extend the existing `YubicoId`/IDV/passkey business logic from this repo** — that logic isn't source-present. But you **can add brand-new Pigeon methods and brand-new native glue code, next to the existing glue, that talk directly to a different backend (SIROS ID) using libraries already vendored into this module** (Retrofit/OkHttp/kotlinx.serialization on Android; the module can add `URLSession`-based Swift networking or a new SwiftPM dependency on iOS). This would NOT be "extending the YubicoSDK" — it would be new code living alongside it in the same plugin, exposed through new/extended Pigeon-generated Dart APIs. Whether this is worth doing here vs. building the OIDC4VCI/VP/R2PS client entirely inside the consuming app (bypassing this plugin) is a design call for the synthesis phase — see §3.

---

## 1. What actually implements the named methods today

### 1.1 Call chain is thin glue → closed SDK singleton, on both platforms

The Pigeon schema `pigeons/yubico_api.dart:485-621` defines `abstract class YubicoHostApi` with `initializeSdk`, `startIdvSession`/`startIdvLiveness`/`startIdvDocumentScan`/`startIdvNfcScan`/`triggerIdvVerification`/etc., `parseOnboardingToken`, `startYubicoIdFlow`, `provideYubicoIdv`, `provideYubicoIdCredentials`, `fetchCreateOptions`, `createSoftwarePasskey`, `createHardwarePasskey`, `listHardwareCredentials`, `listCredentials`, `assertCredential`, and more (pigeons/yubico_api.dart:486-621). This generates:
- `lib/src/pigeon/yubico_api.g.dart` (Dart client)
- `android/src/main/kotlin/com/yubico/yubico_sdk_flutter/pigeon/YubicoApi.g.kt` (Kotlin host interface)
- `ios/yubico_sdk_flutter/Sources/yubico_sdk_flutter/YubicoApi.g.swift` (Swift host interface)

`lib/yubico_sdk_flutter.dart:72-102` shows the public Dart facade (`YubicoSdkFlutter.startYubicoIdFlow`, `.provideYubicoIdv`, `.provideYubicoIdCredentials`) simply forwarding to `YubicoSdkFlutterPlatform.instance` (method-channel/pigeon default impl in `lib/yubico_sdk_flutter_method_channel.dart`), with no business logic in Dart at all.

**Android** — `android/src/main/kotlin/com/yubico/yubico_sdk_flutter/YubicoSdkFlutterPlugin.kt` implements `YubicoHostApi` directly:
- `startYubicoIdFlow` (line 473-499): normalizes args, then `sdk.yubicoId.startFlow(normalizedToken, normalizedDeviceId)` (line 485) inside `runBlocking`, maps `SDKResult.Success/Failure` to the Pigeon `YubicoIdResponseData`/`YubicoHostApiError`. No HTTP/OAuth code of its own.
- `provideYubicoIdv` (line 501-535): calls `sdk.yubicoId.provideIdv(normalizedToken, normalizedIdvData)` (line 521).
- `provideYubicoIdCredentials` (line 537-569): calls `sdk.yubicoId.provideCredentials(normalizedToken, normalizedCredentialUuid, credential.toJson())` (line 551-555).
- `parseOnboardingToken` (line 451-471): `sdk.passkeys.parseOnboardingToken(rawQrCode)` (line 455).
- `fetchCreateOptions` (line 571-598): `sdk.enrollment.fetchCreateOptions(token, oneTimeCode)` (line 581).
- `createSoftwarePasskey`/`createHardwarePasskey`/`listHardwareCredentials`/`assertCredential`/`listCredentials`/`getYubiKeyInfo`/`getPinStatus`/etc. all delegate to `sdk.passkeys.*` (e.g. line 611 `sdk.passkeys.createSoftwareKey(...)`, line 648 `sdk.passkeys.createHardwareKey(...)`, line 704 `sdk.passkeys.listHardwareCredentials(...)`, line 1110 `sdk.passkeys.getAssertion(...)`, line 1081 `sdk.passkeys.listCredentials(rpFilter)`).
- `sdk` here is `com.yubico.sdk.YubicoSDK.getInstance()` (imported at line 14: `import com.yubico.sdk.YubicoSDK`), whose `.yubicoId`, `.idv`, `.passkeys`, `.enrollment` sub-APIs are typed against classes imported from `com.yubico.sdk.core.*`, `com.yubico.sdk.enrollment.*`, `com.yubico.sdk.passkeys.*` (imports at lines 14-69) — **all of which resolve only to the vendored AAR classfiles in `libs/`**, not to any source in this repo.
- `android/src/main/kotlin/com/yubico/yubico_sdk_flutter/idv/IdvSessionController.kt` is a real, source-present state machine (Idle → Initialising → ReadyForLiveness → FaceEnrollment/FaceVerification → ReadyToScanDocument → DocumentScan → DocumentScanPreview → NfcWait → ReadyToVerify → Finalising → Complete/Error), but every single state transition's actual work is one call into `sdk().idv.*` (`initSession`, `processFaceEnrollmentFrame`, `processDocumentScanFrame`, `readNfcPassport`, `finaliseVerification`, `performReauth`, etc. — lines 94, 481-485, 277, 328, 375, 519). The controller only orchestrates UI state; it never talks to a network or a biometric matcher directly.

**iOS** — `ios/yubico_sdk_flutter/Sources/yubico_sdk_flutter/YubicoSdkFlutterPlugin.swift` mirrors this exactly:
- `startYubicoIdFlow` (line 269-281): `sdk.yubicoId.startFlow(token:deviceId:)` (line 276).
- `provideYubicoIdv` (line 283-298): `sdk.yubicoId.provideIdv(token:idvData:)` (line 293).
- `provideYubicoIdCredentials` (line 300-323): `sdk.yubicoId.provideCredentials(token:credentialUuid:credentialJson:)` (line 315-318).
- `parseOnboardingToken` (line 260-267): `sdk.passkeys.parseOnboardingToken(raw:)` (line 262).
- `fetchCreateOptions`/`createSoftwarePasskey`/`createHardwarePasskey`/`listHardwareCredentials`/`assertCredential`/`listCredentials` etc. all delegate to `sdk.enrollment.*` / `sdk.passkeys.*` (lines 335, 358, 386, 429, 675/701, 701).
- `sdk` is `yubicoSDKShared()` returning a `YubicoSDK` instance imported from `import YubicoSDK` (line 7) — an SwiftPM `.binaryTarget` pointing at `ios/Frameworks/YubicoSDK.xcframework` (confirmed in `ios/yubico_sdk_flutter/Package.swift:34-46`). `YubicoSDKCore`, `YubicoSDKEnrollment`, `YubicoSDKIDV`, `YubicoSDKPasskeys` are separate `.binaryTarget`s, each an xcframework in `ios/Frameworks/`.
- Two TODOs in the iOS file (lines 450-451, 467-468, 520-521) explicitly note the plugin currently **bypasses** the native SDK's own hardware-key metadata store and uses a local `UserDefaults`-backed shim (`FlutterEnrolledHardwareKeyStore`, lines 1033-1136) "when the iOS SDK metadata store is fixed" — direct evidence the plugin authors treat the compiled SDK as an external, not-fully-controllable dependency they work around rather than fix.

### 1.2 Backend/vendor identity

- `example/.env.local.example` contains `PRIVATE_ID_API_KEY`, `PRIVATE_ID_BASE_URL`, `PRIVATE_ID_PRODUCT_GROUP_ID`, `PRIVATE_ID_REDIRECT_URL`, `PRIVATE_ID_CALLBACK_URL`, `PRIVATE_ID_USER_ID` — these map 1:1 onto `IdvSessionConfigData`'s `apiKey`/`baseUrl`/`productGroupId`/`redirectUrl`/`callbackUrl` fields (pigeons/yubico_api.dart:365-376). **The IDV vendor is "Private ID"** (confirmed again by git history: commit `ff945a4 chore(and): updated native SDKs v0.0.39 - PrivateID v0.5.0`, and by `libprivid_fhe_ultra.so` / `privid_fhe_ultra.xcframework` — "FHE" = fully-homomorphic-encryption-based biometric matching, Private ID's core IP). The Keystone/UltraPass-branded AARs (`keystone-ultrapass-*`) and `PassportScan.xcframework`/`KeyStoneSDKNfc.xcframework` are Private ID's document/NFC/liveness capture components, vendored a second time from a Private ID SDK checkout referenced by path in `scripts/update_native_sdk_android.sh:81` (`${SDK_ROOT}/sdk-idv/libs/keystone-ultrapass`).
- `libs/facetec-sdk-10.0.15.aar` (FaceTec Inc.'s liveness SDK) is present but **not** listed among the AARs `scripts/update_native_sdk_android.sh` downloads or mirrors — it appears to be manually vendored, likely a dependency *of* the native SDK's IDV module (FaceTec 3D liveness) rather than something referenced by name from this plugin's own Kotlin code.
- The enrollment/`YubicoId` flow talks to a Yubico-owned backend: `ios/.../YubicoSdkFlutterPlugin.swift:1483,1493` hard-codes fallback URLs `https://enrollment.dev.yubico.com/credentials/v1/` and `https://enrollment.dev.yubico.com/credentials/token/v1/` for the `dev` environment when no override is supplied — i.e. Yubico's own "enrollment" service, separate from Private ID.
- Net: `startYubicoIdFlow`/`provideYubicoIdv`/`provideYubicoIdCredentials` are backed by a **Yubico-proprietary backend** (the "YubicoId" enrollment/credential-issuance service), with the IDV step of that flow delegating internally to **Private ID** as a subcontracted biometric-verification vendor. None of this is an OpenID4VCI/VP-compliant protocol — it is Yubico's own bespoke JSON/REST protocol, opaque past the SDK boundary.

---

## 2. What's in `libs/` / `ios/Frameworks/`

| Artifact | Type | Contents (from `unzip -l`) | Source |
|---|---|---|---|
| `libs/sdk-release.aar` | AAR, 8 KB | `classes.jar` (tiny facade, likely just the `YubicoSDK` singleton) | `Yubico/hippo-yubico-wallet-sdk-android` release |
| `libs/sdk-core-release.aar` | AAR | `classes.jar` (~95 KB compiled Kotlin, `com.yubico.sdk.core.*`) | same |
| `libs/sdk-enrollment-release.aar` | AAR | `classes.jar` (~358 KB, `com.yubico.sdk.enrollment.*`, incl. `YubicoId*` classes) | same |
| `libs/sdk-idv-release.aar` | AAR | `classes.jar` (~294 KB, `com.yubico.sdk.idv.*`) | same |
| `libs/sdk-passkeys-release.aar` | AAR | `classes.jar` (~400 KB, `com.yubico.sdk.passkeys.*`) + a Credential Provider `AndroidManifest.xml`/`res/xml/yubico_credential_provider.xml` | same |
| `libs/keystone-ultrapass-native-release.aar` | AAR | `classes.jar` (87 KB) + **native `.so`**: `libnativelib.so`, `libprivid_fhe_ultra.so` (9.1 MB arm64 / 6.8 MB armeabi-v7a), `libc++_shared.so` | Private ID, mirrored from a separate native-SDK checkout (`sdk-idv/libs/keystone-ultrapass`) |
| `libs/keystone-ultrapass-nfc-release.aar` | AAR | `classes.jar` + `libopenjpeg.so` (JPEG2000 decode for passport photos, all ABIs) | Private ID |
| `libs/keystone-ultrapass-orchestration-release.aar`, `keystone-ultrapass-sdk-release.aar` | AAR | `classes.jar` only | Private ID |
| `libs/facetec-sdk-10.0.15.aar` | AAR, 8.8 MB | `classes.jar` + native `libaa08.so`/`libPhoenixAndroid.so` (obfuscated names — FaceTec's own liveness engine) + encrypted asset blobs under `assets/com/facetec/sdk/...` | FaceTec Inc., vendored manually (not in the update script's managed list) |
| `ios/Frameworks/YubicoSDK.xcframework`, `YubicoSDKCore/Enrollment/IDV/Passkeys.xcframework` | Binary xcframeworks | Compiled Swift/ObjC binaries, no source | `Yubico/hippo-yubico-wallet-sdk-ios` release |
| `ios/Frameworks/KeyStoneSDK.xcframework`, `KeyStoneSDKNfc.xcframework`, `PassportScan.xcframework`, `privid_fhe_ultra.xcframework` | Binary xcframeworks | Compiled binaries | Private ID, bundled into the same iOS release zip (`YubicoSDKBinaryDistribution.zip`) |
| `ios/Frameworks/OpenSSL.xcframework` | Binary xcframework | Prebuilt OpenSSL (headers present, e.g. `Headers/asn1.h`) | Third-party, bundled in the same release |

**No Rust core / no `r2ps`-style crate anywhere in this repo.** There is no Cargo, no `.dll`/`.so` that looks like a Rust WSCD-style component, and no source for any of the above — everything under `libs/` and `ios/Frameworks/` is compiled output. `CHANGELOG.md` is an unedited stub (`## 0.0.1 / * TODO: Describe initial release.`) and never documents what's vendored; `README.md` only documents the *scripts* to refresh the binaries (`scripts/update_native_sdk.sh`, `update_native_sdk_android.sh`, `update_native_sdk_ios.sh`), which download from two sibling **private** GitHub repos:
- `Yubico/hippo-yubico-wallet-sdk-android` (Android native SDK source — not checked out here)
- `Yubico/hippo-yubico-wallet-sdk-ios` (iOS native SDK source — not checked out here)

`scripts/update_native_sdk_android.sh:81-100` additionally shows the Keystone/Private-ID AARs are mirrored from a **third** location, an environment-variable-configurable native-SDK-android checkout's own `sdk-idv/libs/keystone-ultrapass/` folder — i.e., even the Android native SDK repo itself treats Private ID's SDK as a vendored third-party binary, not something it builds from source.

---

## 3. Feasibility assessment (critical deliverable)

**Verdict: (b) partially extensible — cleanly split between a fully-editable "shell" (this repo) and a fully-opaque "core" (the sibling native SDK repos + third-party vendor binaries).**

What's true at the same time:

1. **The existing `YubicoId`/IDV/passkey protocol logic cannot be modified or extended from this repo.** Everything that would need to change to make `startYubicoIdFlow`/`provideYubicoIdv`/`provideYubicoIdCredentials` (or the IDV session state machine, or passkey creation) speak OpenID4VCI/OpenID4VP or R2PS instead of Yubico's/Private ID's proprietary protocols is compiled into `com.yubico.sdk.*` classes (Android) or the `YubicoSDK*.xcframework` binaries (iOS), sourced from two separate private repos (`hippo-yubico-wallet-sdk-android`/`-ios`) that are **not present in this checkout**. You cannot recompile, patch, or subclass around their internal HTTP client, token handling, or crypto — Kotlin/Swift interop only exposes their public API surface (`sdk.yubicoId.startFlow(...)`, etc.), and that surface is intentionally narrow (three opaque round-trip calls per pigeons/yubico_api.dart:516-527). There's no plugin point to redirect those calls at a different base URL beyond `SdkConfigData.yubicoIdBaseUrlOverride` (pigeons/yubico_api.dart:154) — which only changes *where* the same proprietary protocol is sent, not *what* protocol is spoken.

2. **The plugin shell around that core is ordinary, fully-editable Kotlin/Swift/Dart source, and it is not thin in the "nothing to build on" sense — it's thin in the "no business logic yet" sense.** `YubicoSdkFlutterPlugin.kt`/`.swift` are ~1400-2300 lines respectively of real glue code (coroutine/Task orchestration, Pigeon (de)serialization, FlutterApi event forwarding, camera/platform-view wiring for IDV, PIN/reauth provider callbacks). Adding wholly new Pigeon methods (e.g. `startCredentialIssuance(offerUri)`, `presentCredential(requestUri)`, `provisionRemoteKey(...)`) is mechanically the same pattern already used for every existing method: extend `pigeons/yubico_api.dart`, regenerate with `scripts/generate_pigeon.sh`, implement the new `YubicoHostApi` methods in Kotlin/Swift by calling **new code you write**, not the vendored SDK.
3. **The Android module already declares production HTTP/crypto/serialization dependencies that a new implementation could reuse directly** — Retrofit, OkHttp (with logging interceptor), kotlinx.serialization-json, BouncyCastle (`bcpkix-jdk15on`), all declared in `android/build.gradle.kts:104-125` (also centralized in `android/gradle/yubico-wallet-libs.versions.toml`). These are currently `implementation` deps supplied so the **closed AAR classes** can resolve their own Retrofit/OkHttp/serialization references at runtime (the AARs are `compileOnly`, so the host module must supply matching runtime jars) — but Gradle does not restrict them to that use; new Kotlin code added to this same module can `import retrofit2.*` / `import okhttp3.*` and use them for a fresh OpenID4VCI/VP HTTP client with zero new dependency wiring.
4. **iOS has less to reuse.** `ios/yubico_sdk_flutter/Sources/YubiKit/YubiKit.swift` is a nearly-empty placeholder shim (a handful of empty class stubs: `YKFNFCConnection`, `YKFFIDO2Session`, etc. — no real YubiKit integration). `ios/Frameworks/OpenSSL.xcframework` is present but only linked for the native SDK's own use. There is no Retrofit-equivalent already wired for iOS — a new OpenID4VCI/VP client there would use `URLSession` directly (standard, no new dependency needed) or add a new SwiftPM dependency to `ios/yubico_sdk_flutter/Package.swift`, which is straightforward given the existing `.binaryTarget`/`.target` structure.
5. **R2PS/WSCD-style remote key provisioning (OPAQUE + P256 signing) has no counterpart anywhere in this repo.** `androidx.credentials`/`androidx.biometric`/`androidx.security:security-crypto` (Android) and the platform Keychain/Secure Enclave (implied by `keychainAccessGroup`/`appGroupIdentifier` config fields, pigeons/yubico_api.dart:151-152) are the only key-storage primitives visible, used exclusively by the closed passkey SDK. No OPAQUE library, no P256 signing helper, nothing R2PS-shaped exists in source form here — this would be 100% new code (Android: e.g. add a JVM OPAQUE lib + `java.security`/BouncyCastle-based P256 signing, since BouncyCastle is already a dependency; iOS: `CryptoKit`'s P256 support is built into the platform and needs no new dependency, but OPAQUE itself would need a new library).

**Consequence for the design-synthesis phase:** building OpenID4VCI/OpenID4VP/R2PS support *inside* this plugin is mechanically possible (new Pigeon methods + new native glue code that reuses Retrofit/OkHttp on Android and URLSession on iOS) but delivers **no reuse of the existing YubicoId/IDV/passkey code paths** — it would be net-new protocol-client code living in the same repo/module as, but architecturally parallel to, the existing closed-SDK-backed methods. The only things actually shared would be: the Pigeon/plugin-channel plumbing pattern, the module's existing HTTP dependency declarations (Android), and possibly the existing `androidx.credentials`/passkey machinery if SIROS ID's WebAuthn/passkey story can reuse `createSoftwarePasskey`/`createHardwarePasskey`/`assertCredential` as-is (those ARE full, working FIDO2/WebAuthn primitives against the closed SDK, and per the mandate's framing are a separate concern from the credential-issuance flow — they may well be directly reusable for SIROS ID's WebAuthn passkey feature without any new code, since WebAuthn is a standard and the create/assert options already flow through this plugin as data, not tied to Yubico's backend by protocol). Whether to (a) add SIROS ID methods into this plugin's native layer, or (b) implement OpenID4VCI/VP/R2PS entirely in the consuming Dart/Flutter app via `package:http`/`dio` + a pure-Dart or platform-channel-free crypto library (bypassing this plugin altogether) is a tradeoff for the next phase — but it is **not** blocked by the closed SDK either way, since no path here requires modifying the closed SDK's behavior.

---

## 4. Existing native/HTTP/crypto dependencies (reuse vs. build fresh)

### Android (`android/build.gradle.kts:101-143`, `android/gradle/yubico-wallet-libs.versions.toml`)
Already present, could be reused as-is for a new OpenID4VCI/VP/R2PS implementation:
- **HTTP**: `com.squareup.retrofit2:retrofit:3.0.0`, `converter-kotlinx-serialization`, `converter-gson:2.11.0`, `com.squareup.okhttp3:logging-interceptor:4.12.0`
- **Serialization**: `org.jetbrains.kotlinx:kotlinx-serialization-json:1.9.0`
- **Crypto**: `org.bouncycastle:bcpkix-jdk15on:1.65` (X.509/PKIX — usable for P256/COSE work), `com.upokecenter:cbor:4.5.6`, `com.augustcellars.cose:cose-java:1.1.0` (COSE, relevant to WebAuthn attestation, could be reused for mdoc/SD-JWT COSE structures if SIROS ID needs them)
- **Secure storage**: `androidx.security:security-crypto:1.1.0-alpha06`
- **Passkeys/FIDO2**: `androidx.credentials:credentials:1.5.0`, `com.yubico.yubikit:core/fido/android/management:3.0.1` (Yubico's **open-source** YubiKit — physical-key FIDO2/PIV, separate from the closed proprietary SDK; this is the one piece of "Yubico SDK" that genuinely is open source and could be extended)
- **MRTD/passport**: `org.jmrtd:jmrtd:0.7.18`, `net.sf.scuba:scuba-sc-android:0.0.18`, `com.github.mhshams:jnbis:1.1.0` — these exist to support the closed IDV SDK's NFC passport reading; not directly relevant to OIDC4VCI/VP but available if SIROS ID ever needs MRTD parsing.
- Would need to be added fresh: an OAuth2/OpenID4VCI/VP client library (none present — Retrofit+kotlinx.serialization are enough to hand-roll one), a CBOR/mdoc (ISO 18013-5) library if mdoc credential format is needed beyond the existing `cbor`/`cose-java` primitives, an OPAQUE implementation for R2PS.

### iOS (`ios/yubico_sdk_flutter/Package.swift`, `ios/yubico_sdk_flutter.podspec`)
- No third-party networking/crypto SwiftPM dependencies declared at all beyond the vendored binary frameworks (`OpenSSL.xcframework` is linked but only for the closed SDK's use).
- `YubiKit.swift` (`ios/yubico_sdk_flutter/Sources/YubiKit/YubiKit.swift`) is an essentially empty placeholder (bare class stubs for `YKFNFCConnection`, `YKFFIDO2Session`, etc.) — not a real integration to build on.
- Everything for a new implementation would be added fresh: `URLSession`-based HTTP (no new dependency needed) or a new SwiftPM package dependency, `CryptoKit` for P256 (built into the platform, free), and a new OPAQUE library for R2PS (none exists on Apple platforms in-repo or in-OS).

### Dart (`pubspec.yaml`)
Only dependency is `plugin_platform_interface: ^2.0.2` plus `pigeon` as a dev dependency. **No `http`/`dio`/`oauth2`/`crypto` packages at all** — any OpenID4VCI/VP logic implemented in Dart (rather than native) would need these added fresh; nothing to reuse here beyond the Pigeon codegen pattern itself.

---

## 5. Versioning / CI

- **`pubspec.yaml:3`**: `version: 0.0.1` — never bumped since project scaffolding.
- **`CHANGELOG.md`**: unedited stub (`## 0.0.1` / `* TODO: Describe initial release.`) — not used as a real changelog in practice.
- **No CI at all**: no `.github/` directory, no workflow files of any kind found in the repo.
- **No git tags**: `git tag` returns nothing.
- **Git history** (`git log --oneline`) shows an active, iterative development style with descriptive commit messages (`feat(mdl): ...`, `fix(ios): ...`, `chore(and): updated native SDKs v0.0.39 - PrivateID v0.5.0`) but no release/tag discipline — the native SDK binaries are updated ad hoc via the `update_native_sdk*.sh` scripts pulling tagged releases from the two sibling private repos, while this plugin repo itself is not tagged/versioned in lockstep.
- Given there's no pub.dev publication and no CI, the consuming app almost certainly depends on this package via a **git dependency** (branch or commit pin) in its `pubspec.yaml`, not a semver package version. This means: **a fork/feature-branch of this repo can flow back to the consuming app simply by changing its pinned git ref** (branch/commit SHA) — there's no publish/release pipeline gate to satisfy, but also no compatibility contract enforced by tooling; the consuming app and any modified branch of this SDK would need to be kept in sync manually. If new Pigeon methods are added here, `scripts/generate_pigeon.sh` must be re-run and the generated files (`lib/src/pigeon/yubico_api.g.dart`, the Kotlin/Swift `.g.kt`/`.g.swift` counterparts) committed, matching the existing convention documented in `docs/PROJECT_STRUCTURE.md:39-51`.

---

## Key file references

- `pigeons/yubico_api.dart:365-376,486-621` — Pigeon `YubicoHostApi` definition, incl. all methods named in the mandate and `IdvSessionConfigData` (Private ID config fields).
- `lib/yubico_sdk_flutter.dart:72-102` — Dart public facade, pure delegation.
- `android/src/main/kotlin/com/yubico/yubico_sdk_flutter/YubicoSdkFlutterPlugin.kt:14-69` (imports of closed `com.yubico.sdk.*` classes), `:473-598` (`startYubicoIdFlow`/`provideYubicoIdv`/`provideYubicoIdCredentials`/`fetchCreateOptions`), `:600-1135` (passkey methods).
- `android/src/main/kotlin/com/yubico/yubico_sdk_flutter/idv/IdvSessionController.kt` — full IDV UI state machine, every transition backed by `sdk().idv.*`.
- `ios/yubico_sdk_flutter/Sources/yubico_sdk_flutter/YubicoSdkFlutterPlugin.swift:269-343` (YubicoId/enrollment methods), `:450-534` (TODO comments admitting the plugin works around native SDK gaps with a local shim), `:1479-1497` (hardcoded Yubico `enrollment.dev.yubico.com` URLs).
- `ios/yubico_sdk_flutter/Sources/YubiKit/YubiKit.swift` — near-empty placeholder, not a real integration.
- `ios/yubico_sdk_flutter/Package.swift` — SwiftPM manifest showing every `YubicoSDK*`/`KeyStone*`/`privid_fhe_ultra`/`OpenSSL` target as a `.binaryTarget` (no source).
- `android/build.gradle.kts:101-143`, `android/gradle/yubico-wallet-libs.versions.toml` — existing Retrofit/OkHttp/BouncyCastle/CBOR/COSE/YubiKit dependencies.
- `libs/*.aar` — all compiled (`classes.jar` + `.so`), see table in §2.
- `scripts/update_native_sdk_android.sh`, `scripts/update_native_sdk_ios.sh` — prove the real native SDK source lives in `Yubico/hippo-yubico-wallet-sdk-android` / `-ios` (not this repo), and that Private ID's Keystone/UltraPass AARs are mirrored from yet another (native-SDK-side) checkout.
- `example/.env.local.example` — `PRIVATE_ID_*` env vars confirming the IDV vendor identity.
- `pubspec.yaml`, `CHANGELOG.md` — versioning stubs; no CI (`.github/` absent); no git tags.
