# Slice B — protocol client package (pure Dart, no UI)

Implementation notes for Slice B of the YubiId VC wallet integration effort
(see `docs/integration/04-integration-design.md` §1, §2, §5 Slice B, §6).
Everything lives in the `hippo-yubico-app-user` repo, entirely under a new
local package — no other repo was touched, and (per instructions) nothing
under `lib/features/**` or `lib/app/**` was touched either. This is the only
repo Slice B was scoped to.

## Package location

`hippo-yubico-app-user/packages/vc_wallet_client/` — a standalone, pure-Dart
package (no Flutter dependency), referenced from the root app via a `path:`
dependency:

```yaml
# hippo-yubico-app-user/pubspec.yaml
dependencies:
  ...
  vc_wallet_client:
    path: packages/vc_wallet_client
```

That one line is the only change made to the root `pubspec.yaml`. No file
under `lib/features/**` or `lib/app/**` was touched.

### File layout

```
packages/vc_wallet_client/
  pubspec.yaml
  analysis_options.yaml
  lib/
    vc_wallet_client.dart              # public API barrel
    src/
      client/vc_wallet_client.dart     # VcWalletClient (main entry point)
      dpop/dpop_proof_generator.dart   # RFC 9449 DPoP proofs
      http/vc_http_client.dart         # http.Client wrapper + nonce retry
      http/endpoints.dart              # base-URL + path join helper
      issuance/issuance_service.dart   # pre-auth + PAR/PKCE/auth-code flows
      presentation/presentation_service.dart  # OpenID4VP flow
      presentation/dcql_matcher.dart   # DCQL query matching
      jose/base64url.dart              # base64url helpers
      jose/ec_p256_key_pair.dart       # EC P-256 keygen/signing (pointycastle-backed)
      jose/jws.dart                    # minimal compact-JWS build/decode
      sdjwt/sd_jwt_parser.dart         # SD-JWT VC -> SdJwtVc parsing
      sdjwt/sd_jwt_presentation_builder.dart  # re-disclosure + KB-JWT
      models/*.dart                    # value types (see "Public API" below)
  test/
    dpop_proof_generator_test.dart
    sd_jwt_parser_test.dart
    dcql_matcher_test.dart
    credential_offer_resolution_test.dart
    pre_authorized_issuance_test.dart
    authorization_code_issuance_test.dart
    presentation_flow_test.dart
    fixtures/sample_sd_jwt.dart
    integration/live_stack_test.dart   # skipped by default, see below
```

## Public API

Everything is exported from `package:vc_wallet_client/vc_wallet_client.dart`.
The main entry point:

```dart
class VcWalletClient {
  static Future<VcWalletClient> create({
    required VcWalletClientConfig config,
    http.Client? httpClient, // injectable for tests/DI; omit for production
  });

  // Issuance (OpenID4VCI)
  Future<CredentialOffer> resolveOffer(String offerUriOrPayload);
  Future<Map<String, dynamic>> fetchIssuerMetadata();
  Future<IssuedCredential> issuePreAuthorized({
    required CredentialOffer offer,
    required String credentialConfigurationId,
    String? txCode,
  });
  Future<AuthorizationRequestHandle> beginAuthorizationCodeIssuance({
    required String credentialConfigurationId,
  });
  Future<IssuedCredential> completeAuthorizationCodeIssuance({
    required AuthorizationRequestHandle handle,
    required Uri redirectResponseUri,
  });

  // Presentation (OpenID4VP)
  Future<PresentationRequest> resolvePresentationRequest(String requestUriOrDeepLink);
  Future<PresentationResult> submitPresentation({
    required PresentationRequest request,
    required IssuedCredential credential,
  });
  Future<PresentationResult> pollPresentationResult(String sessionId);

  // Utilities
  static Future<SdJwtVc> parseSdJwtVc(String rawCredential); // re-parse from storage
  void close();
}
```

This matches the shape sketched in design-doc §3's `VerifiableCredentialSdkPort`
closely enough that Slice C's port implementation should be a thin
pass-through: `resolveCredentialOffer` → `resolveOffer`,
`issueCredential` → `issuePreAuthorized`/the two-call authorization_code pair,
`resolvePresentationRequest`/`submitPresentation` map 1:1. Storage
(`listStoredCredentials`, etc.) is intentionally **not** part of this
package — persistence is Slice C's job (`flutter_secure_storage`), and
`IssuedCredential`/`HolderKeyMaterial` are plain, JSON-friendly value types
specifically so Slice C can serialize them directly.

Config:

```dart
class VcWalletClientConfig {
  const VcWalletClientConfig({
    required Uri apigwBaseUrl,
    required Uri verifierBaseUrl,
    required String clientId,
    required Uri redirectUri,
  });
}
```

No sirosid-dev-specific value is hardcoded anywhere in the package — the
fixed values from the design doc / Slice A
(`scope`/`credential_configuration_id` = `employee_id`,
VCT = `urn:yubico:employeeid:1`, `client_id` = `yubiid-app`,
`redirect_uri` = `app.yubiid.vc://callback`, verifier base URL port `9001`,
apigw base URL port `9003` — the latter two per
`docs/integration/05a-implementation-notes-backend.md`) only appear as
example/test defaults in `test/integration/live_stack_test.dart` and in code
comments, exactly as instructed.

Domain models exported: `CredentialOffer` (+ `PreAuthorizedCodeGrant`,
`AuthorizationCodeGrant`), `IssuedCredential`, `SdJwtVc` (+
`SdJwtDisclosure`), `HolderKeyMaterial`, `AuthorizationRequestHandle`,
`PresentationRequest`, `PresentationResult` (+ `PresentationStatus` enum),
`TokenResponse`, and the exception hierarchy
`VcWalletClientException`/`VcHttpException`/`VcProtocolException`.

## Dependency choices

```yaml
dependencies:
  cryptography: ^2.9.0   # SHA-256 hashing only (see below)
  http: ^1.2.2
  meta: ^1.15.0
  pointycastle: ^4.0.0    # EC P-256 keygen + ECDSA signing
  uuid: ^4.5.1
```

- **`http`**, not `dio`: the package only ever does simple
  request/response calls (no interceptors, multipart, or streaming), and
  `package:http` ships a built-in `MockClient` (`http/testing.dart`) that is
  a very direct fit for the "mock HTTP responses" testing style the design
  doc asks for — no extra mocking package needed. All tests use it.
- **`uuid`**: `jti` (DPoP + OpenID4VCI proof JWTs) and OAuth `state`. PKCE
  `code_verifier` uses `dart:math`'s `Random.secure()` directly (no need for
  a dedicated PKCE package for a few lines of RFC 7636 code).
- **No JOSE library** (`jose`/`dart_jsonwebtoken`) was added. This package
  only needs a narrow slice of JWS functionality — build ES256 JWTs it signs
  itself, and decode-but-not-verify JWTs it receives — implemented directly
  in ~80 lines (`lib/src/jose/jws.dart` + `ec_p256_key_pair.dart`). Both of
  the mainstream Dart JOSE libraries end up depending on `pointycastle` for
  their actual EC math anyway (see below), so pulling one in wouldn't have
  avoided a crypto dependency — it would only have added surface area
  (multi-algorithm support, JWK parsing for algorithms this package doesn't
  use, etc.) without benefit.

### Important deviation: `cryptography` cannot do the EC/ECDSA work — `pointycastle` was added

The design doc (§2, §5 Slice B) says to reuse `cryptography: ^2.9.0` (already
an app dependency) for "EC P-256 keypair generation, ECDSA sign/verify" and
explicitly avoid adding a second crypto library. **This premise turned out to
be incorrect**, discovered while implementing (verified against the
`cryptography` package's own README and source on GitHub, since no Dart/
Flutter SDK was available in this sandbox to simply try it and see):

- `package:cryptography`'s README states outright: *"we don't have
  implementations of \[Ecdsa.p256/p384/p521\] in pure Dart."*
- Its `DartEcdsa` backend class (`cryptography/lib/src/dart/ecdsa.dart`) —
  the one that's supposed to run on a plain Dart VM with no platform
  channel — has `newKeyPair()`/`sign()`/`verify()` all implemented as
  `throw UnimplementedError()`.
- Real ECDSA signing only happens via the browser's Web Crypto API (web
  platform only) or the separate `cryptography_flutter` plugin, which
  bridges to iOS CryptoKit / Android's `javax.crypto` over platform
  channels — i.e. it requires a running Flutter app on a real
  device/simulator, registered at startup. Neither path exists in a plain
  `dart test` run, which is exactly how this package's own test suite (and
  the design doc's "pure Dart, unit-testable" premise in §2) needs to run.

Continuing to use `package:cryptography` for EC operations as originally
planned would have meant every DPoP proof, OpenID4VCI proof-of-possession
JWT, and SD-JWT key-binding JWT this package builds — i.e. essentially all
of its core logic — would throw at runtime in any context except a real
Flutter app build with `cryptography_flutter` wired up, and **could never be
unit-tested** the way the design doc's Slice B acceptance criteria and task
instructions require.

**Resolution:** `package:pointycastle` (`^4.0.0`) — a long-established,
genuinely pure-Dart, no-platform-channel port of BouncyCastle — was added
and is used exclusively for EC P-256 key generation and ECDSA signing
(`lib/src/jose/ec_p256_key_pair.dart`). `package:cryptography` is still used,
unmodified from the original plan, for SHA-256 hashing (`DPoP`'s `ath` claim,
SD-JWT disclosure digests, the PKCE `S256` code challenge) — its SHA-256
implementation *is* real, pure-Dart code (verified by reading
`cryptography/lib/src/dart/sha256.dart`), so no change was needed there. Net
effect: one new dependency (`pointycastle`) versus the zero-new-crypto-deps
plan in §5 Slice B, isolated entirely behind the `EcP256KeyPair` wrapper
class so nothing else in the package (or a future consumer) depends on
`pointycastle` directly. Signature bytes produced are the raw fixed-length
`r || s` concatenation JWS ES256 requires (no DER re-encoding).

This is worth flagging up to whoever owns `hippo-yubico-sdk-flutter`/the
native SDK too: if any *other* part of the app ever wants pure-Dart P-256
signing (vs. calling into the native SDK's own crypto), the same gap will
apply there.

## Implemented functionality

- **DPoP (RFC 9449)** — `DpopProofGenerator`: one ephemeral EC P-256 key per
  generator instance (one instance per `VcWalletClient`/session, per the
  design doc), producing `dpop+jwt` proofs with embedded `jwk`, `htm`, `htu`
  (query/fragment stripped per §4.2), `iat`, `jti`, optional `nonce`/`ath`.
  `VcHttpClient` transparently retries once on a
  `{"error":"use_dpop_nonce"}` + `DPoP-Nonce` response, and caches the last
  seen nonce so most calls don't need the retry round trip at all.
- **OpenID4VCI issuance**, both entry points, sharing one `POST /credential`
  code path:
  - (a) pre-authorized_code: offer resolution (inline JSON,
    `credential_offer`/`credential_offer_uri` deep-link params, or a direct
    `GET /credential-offer/:uuid`-style URL) → `POST /token` → `POST
    /credential` with an `openid4vci-proof+jwt` bound to `c_nonce`.
  - (b) PAR + PKCE + `authorization_code` + DPoP: `POST /op/par` (S256 PKCE
    challenge, `authorization_details` naming the
    `credential_configuration_id`) → caller opens the returned
    `/authorize?...&request_uri=...` URL in a browser/webview (out of this
    package's scope, by design) → caller captures the redirect and calls
    `completeAuthorizationCodeIssuance` (`state` is checked; `POST /token`
    with `grant_type=authorization_code` + `code_verifier` + DPoP) → same
    `POST /credential` path as (a).
- **SD-JWT VC parsing** (`SdJwtParser`) — splits the compact string into the
  issuer-signed JWT + disclosures, decodes each disclosure
  (`[salt, name, value]`), merges disclosed + always-visible claims into one
  map, and best-effort-checks each disclosure's digest against the issuer
  JWT's `_sd` array (`SdJwtDisclosure.digestMatchesIssuedJwt`) without
  treating a mismatch as fatal — this package trusts issuers vc-apigw itself
  already trusts (go-trust enforcement happens server-side).
- **OpenID4VP presentation** (`PresentationService`) — resolves an
  `openid4vp://` deep link / bare `request_uri` / inline request-object JWT,
  decodes the signed request object (not verified — see "Known gaps"),
  matches the held credential against `dcql_query` by format
  (`dc+sd-jwt`/`vc+sd-jwt`) + `vct_values` (`DcqlMatcher`), builds the
  presentation string (v1 always re-discloses every claim, per instructions
  — no holder-side selective filtering), signs a KB-JWT when the request
  supplies a nonce/audience, `POST`s to `response_uri`, and exposes
  `pollPresentationResult` for `/poll/:session_id`.

## Deviations from the design doc (besides the crypto dependency above)

1. **`vp_token`/submission field shape is a best-effort reconstruction, not
   verified against a live verifier.** Report 01 §3 documents the overall
   flow (`POST /verification/oidc-direct_post` with `vp_token` +
   "`presentation_submission`") but not vc-verifier's exact request schema —
   no OpenAPI spec exists for it (report 01 explicitly notes this gap for
   the client API surface generally). Since DCQL is the only supported query
   language (no Presentation Exchange, per report 01 §3 — `PresentationDefinition`
   is commented out server-side), and `vp_token` is documented as
   `map[string][]string` keyed by DCQL query id, this package `POST`s a
   form-encoded body with a single `vp_token` field (a JSON-encoded
   `{queryId: [presentationString]}` map) and `state` — it does **not** send
   a separate `presentation_submission` field, since one isn't meaningful
   for a DCQL-only flow. **This is the single highest-risk unverified
   assumption in this slice** — flagged prominently in
   `test/integration/live_stack_test.dart` as the first thing to check once
   a live stack is reachable.
2. **PAR (`POST /op/par`) is not sent with a DPoP proof.** Report 01 doesn't
   state whether PAR itself requires DPoP (only that "authorization-code
   flow requires DPoP" and "token type is always DPoP", which this package
   reads as covering `/token` and `/credential`). Standard OAuth DPoP usage
   binds the token at `/token`, not at PAR, so no DPoP header is sent on
   `/op/par` by default. If the live stack rejects this, adding it is a
   one-line change in `IssuanceService.beginAuthorizationCodeIssuance`
   (pass `useDpop: true` to the existing `postForm` call).
3. **No issuer/request-object signature verification.** SD-JWT VCs received
   from apigw and OpenID4VP request objects received from vc-verifier are
   parsed but not cryptographically verified against issuer/verifier signing
   keys in this v1. Per the design doc, trust enforcement
   (go-trust/AuthZEN) already happens server-side before a wallet ever sees
   these; this package's models still expose enough (`SdJwtDisclosure
   .digestMatchesIssuedJwt`, the raw `requestObjectJwt` string) for a
   fast-follow to add verification (fetch `/jwks`, verify ES256) without
   changing the public API shape.
4. **Holder-binding key handling** isn't explicitly specified in the design
   doc's port sketch (§3), since that sketch is Slice C's concern. This
   package generates one holder-binding EC P-256 key per issued credential
   and returns it as `IssuedCredential.holderKey` (a plain, JSON-serializable
   `HolderKeyMaterial`) so Slice C can persist it alongside the credential
   via `flutter_secure_storage` and pass it back into `submitPresentation`
   for KB-JWT signing — no storage happens inside this package itself.

Everything else (scope name, VCT, client_id, redirect_uri, credential
format, claim names) matches Slice A's implementation
(`05a-implementation-notes-backend.md`) exactly.

## Test results

**No live sirosid-dev stack, and no Dart/Flutter SDK at all, were available
in this sandbox** (`dart`, `flutter` were both absent from `PATH` and not
found anywhere on disk; confirmed with repeated `find`/`where` searches).
`dart pub get` / `dart test` could not actually be executed or their results
observed — this is a real limitation of this environment, not a decision.

What was done instead to compensate:

- Every dependency's exact API (constructor signatures, method names,
  nullability) used by this package was individually verified against the
  actual published source on GitHub/pub.dev via web fetches, rather than
  from memory — this is what surfaced the `cryptography`
  pure-Dart-ECDSA gap above, and confirmed the exact
  `pointycastle` API (`ECKeyGenerator`, `ECDSASigner`, `ECPrivateKey`/
  `ECPublicKey`/`ECPoint`/`ECSignature`, `FortunaRandom`,
  `ParametersWithRandom`, etc.) used in `ec_p256_key_pair.dart`.
- Every source file was manually re-read end to end for syntax/type
  correctness (import correctness, brace/paren balance, nullability,
  correct use of `async`/`await`, matcher usage in tests — e.g. tests that
  assert an `async` function throws use `expectLater(future, throwsA(...))`
  against the `Future` itself, not a wrapping closure, since a closure that
  calls an `async` function never synchronously throws to the caller).
- One real bug was caught this way and fixed:
  `List<int>.filled(n, 0)..addAll(bytes)` in the original
  `ec_p256_key_pair.dart` draft — `List.filled` produces a **fixed-length**
  list by default, so `addAll` on it would have thrown `UnsupportedError` at
  runtime. Fixed to `[...List<int>.filled(n, 0), ...bytes]`.

Test suite written (all under `packages/vc_wallet_client/test/`, runnable via
plain `dart test` once a Dart SDK is available — no Flutter/device needed
since the package has no Flutter dependency):

| File | Covers |
|---|---|
| `dpop_proof_generator_test.dart` | DPoP proof shape (`typ`, `alg`, embedded `jwk`, `htm`, `htu` minus query/fragment, `iat`, `jti`), `nonce`/`ath` inclusion, same key reused across proofs from one generator |
| `sd_jwt_parser_test.dart` | Parsing a hand-built sample Employee ID SD-JWT (all 6 claims, `vct`/`iss`/`iat`/`exp`, `cnf.jwk`), disclosure digest matching, re-serialization round-trip, unsupported `_sd_alg` / empty-string error cases |
| `dcql_matcher_test.dart` | DCQL matching by format + `vct_values`, requested-claim-path extraction, non-SD-JWT formats ignored, absent `credentials`/`vct_values` handling |
| `credential_offer_resolution_test.dart` | Inline JSON, `credential_offer` deep-link param, `credential_offer_uri` fetch, direct offer-URL GET |
| `pre_authorized_issuance_test.dart` | Full `/token` → `/credential` round trip against a mocked `http.Client` (`MockClient`), DPoP headers present, `Authorization: DPoP <token>`, use_dpop_nonce retry, missing-tx_code error |
| `authorization_code_issuance_test.dart` | `/op/par` request shape (PKCE challenge, `authorization_details`), authorize-URL construction, `/token` exchange with `code_verifier`, CSRF `state`-mismatch and `error=` redirect handling |
| `presentation_flow_test.dart` | Deep-link → request-object resolution, DCQL-matching `vp_token` construction (verified segment count / trailing KB-JWT), `_default` fallback, vct-mismatch error, non-2xx → failed, poll status mapping |
| `test/integration/live_stack_test.dart` | **Skipped by default** (`skip: '...'` on every test) — exercises the same flows against a real `make up VC=yes` stack; documents exactly what to substitute (a real credential offer, a real OpenID4VP deep link) and calls out the `vp_token` field-shape assumption as the first thing to validate |

Since none of this could actually be executed, these results should be
treated as "should pass, pending a real `dart test` run" rather than
confirmed-green — flagging this clearly for whoever picks this up next with
a working Dart/Flutter toolchain. Recommended first step: run `dart pub get
&& dart test` inside `packages/vc_wallet_client/` before building Slice C on
top of this.

## Root `pubspec.yaml` change

Only the one dependency line was added (see "Package location" above); no
other line in the root `pubspec.yaml` was touched, and nothing under
`lib/features/**`/`lib/app/**` was touched, per instructions.
