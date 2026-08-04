# Phase 0 — Repo Setup Status

Goal: make all repos needed for the SIROS ID ↔ YubiId integration available
locally so Phase 1 analysis agents can read real source instead of docs/guesses.

## Cloned successfully

| Repo | Location | Remote | Branch @ commit | Notes |
|---|---|---|---|---|
| go-wallet-backend | `dev/siros/go-wallet-backend` | https://github.com/sirosfoundation/go-wallet-backend.git | `main` @ `045eca5` (2026-08-03) | Wallet backend API (client :8080, admin :8081) |
| go-trust | `dev/siros/go-trust` | https://github.com/sirosfoundation/go-trust.git | `main` @ `c2fb80a` (2026-08-01), tag `v0.9.3` | AuthZEN trust PDP |
| vc | `dev/siros/vc` | https://github.com/SUNET/vc.git | `main` @ `0307125c` (2026-08-03) | vc-issuer/verifier/apigw/registry (OpenID4VCI/VP) |
| wallet-common | `dev/siros/wallet-common` | https://github.com/sirosfoundation/wallet-common.git | `release/sirosid` @ `433a8f1` (2026-07-03), tag `v0.5.0-sirosid.12` | Shared TS types — on the sirosid release branch, not `main`, matching sirosid-dev README's note that wallet-common/wallet-frontend track `release/sirosid` |
| hippo-yubico-sdk-flutter | `dev/yubico/hippo-yubico-sdk-flutter` | git@github.com:yubico/hippo-yubico-sdk-flutter.git (private, SSH) | `main` @ `ef3565e` (2026-06-30) | Private SDK the hippo app delegates all backend/credential/passkey calls to. Contains `lib/`, `android/`, `ios/`, `libs/`, `pigeons/` (Flutter pigeon codegen — implies a native/platform-channel bridge, not pure Dart) |

## Not cloned (not needed yet)

| Repo | Reason |
|---|---|
| wallet-frontend | Web wallet UI — not directly relevant to a mobile app integration; skip unless Phase 1 backend analyst needs it as an API-usage reference |
| hippo-yubico-design-framework-flutter | UI-only design system dependency; only needed if an implementation agent must get the app building end-to-end. Defer until Phase 3 if needed. |

## Auth / access notes

- Public `sirosfoundation`/`SUNET` repos cloned over HTTPS with no issues.
- Private `yubico/hippo-yubico-sdk-flutter` cloned over SSH successfully — user confirmed SSH access was available.
- No VPN or additional allowlisting was required for any clone.

## Flag for Phase 1

`hippo-yubico-sdk-flutter` contains `pigeons/` (Flutter's Pigeon codegen tool, used to generate type-safe platform channel bindings) plus `android/` and `ios/` native project directories — this strongly suggests the SDK is a Dart-facing wrapper over native Android (Kotlin/Java) and iOS (Swift) implementations, not a pure-Dart HTTP client. The `yubico-sdk-analyst` agent should confirm this and locate the actual native source (likely under `android/src` / `ios/Classes` or similar) to determine what currently backs `YubicoIdSdkPort`.
