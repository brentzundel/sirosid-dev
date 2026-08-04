# SIROS ID Backend API Analysis

Synthesized from research across `go-wallet-backend`, `go-trust`, `vc` (SUNET/vc),
`wallet-common`, and `sirosid-dev`'s own orchestration/fixtures. Covers: the
client-facing API surface, OpenID4VCI issuance, OpenID4VP presentation, trust
PDP integration, R2PS/WSCD key provisioning, and what a brand-new wallet client
(e.g. YubiId) needs to bootstrap against this stack.

## 1. Client-facing API surface

`go-wallet-backend` is one Gin-based binary (`cmd/server/main.go`) with
role-based providers, each an `internal/server/providers.go` entry that
registers routes on either the client router (port **8080**) or the admin
router (port **8081**, separate `http.Server`, `internal/server/server.go:378-451`).
Port **8082** is a WebSocket "engine" port (`/api/v2/wallet`) for an
OID4VCI/OID4VP/VCTM protocol session, not plain REST.

### Auth / user endpoints (8080, `internal/api/handlers.go`)
- Public: `POST /user/register-webauthn-begin`/`-finish`, `POST /user/login-webauthn-begin`/`-finish`, `GET /api/v1/tenants/:id/config`.
- Protected (bearer JWT): `GET /user/session/account-info`, `POST /user/session/settings`, `GET`/`POST /user/session/private-data` (ETag-based optimistic concurrency), `DELETE /user/session`, WebAuthn credential management (`/user/session/webauthn/*`), `GET /issuer/all`, `GET /issuer/:id/metadata` (fetches+caches the issuer's `.well-known/openid-credential-issuer`, 5 min TTL), `GET /verifier/all`, `POST /wallet-provider/key-attestation/generate` (see Key Attestation, §4).
- **Caution:** `README.md`'s "API Endpoints" section is stale/inaccurate vs. the actual Gin routes — treat this document and the source (`internal/api/handlers.go`, `internal/server/providers.go`) as ground truth, not the README.
- Dead/unrouted handlers exist in code but aren't wired to any route (`RefreshToken`, `Logout`, `GetIssuerByID`, `WebSocketKeystore`) — don't rely on them.

### Storage endpoints (8080, `/storage/vc/*`)
`GET/POST /storage/vc`, `POST /storage/vc/update`, `GET/DELETE /storage/vc/:credential_identifier`. No `/storage/vp` route exists despite README claiming it.

### New AS (Authorization Server) module — `/auth/*`, only when `cfg.AS.Enabled`
Passkey-first + OIDC session model, described fully in §6 below (auth/session model). This is the actively-developed session/token layer (`docs/new-as.md`) and likely what a new client should target rather than the legacy JWT model.

### AuthZEN trust proxy — `/v1/evaluate`, `/v1/resolve`
Lets an authenticated client ask wallet-backend to evaluate/resolve trust on its behalf against the tenant's go-trust PDP — see §4.

### Admin API (8081) — NOT for end-user clients
Tenant/issuer/verifier/user/invite CRUD (`/admin/tenants`, `/admin/tenants/:id/issuers`, `.../verifiers`, `.../users`, `.../invites`). This is how VC issuer/verifier trust registrations happen (automatically, per sirosid-dev's `make up VC=yes`), and how operators statically configure things — **not** something a mobile wallet client calls itself. Fully documented and code-accurate in `go-wallet-backend/docs/openapi-admin.yaml` (only spec that exists; there is no OpenAPI spec for the 8080 client API).

### Key Attestation (KA) vs. Wallet Instance Attestation (WIA)
- **Key Attestation is real and implemented**: `POST /wallet-provider/key-attestation/generate` — takes client JWKS + OID4VCI nonce (+ optional WSCD `security_properties`), returns a `keyattestation+jwt` proving the wallet generated a given keypair. This is what a client uses for OID4VCI proof-of-possession.
- **WIA (Wallet Instance Attestation) is NOT implemented** — only referenced as "in progress" in `docs/client-id-strategy.md`. Don't design around a WIA endpoint existing today; only KA exists.

## 2. OpenID4VCI credential issuance flow

The actual wallet-facing OpenID4VCI/OAuth2 surface lives in **vc-apigw**
(SUNET/vc repo), not in vc-issuer directly — vc-issuer's HTTP server only
exposes `/health`/`/swagger`; all VCI traffic is proxied through apigw over
internal gRPC. **A wallet client should target vc-apigw's base URL for
everything VCI-related** (`https://apigw.vc.docker:8080` in the dev config;
sirosid-dev exposes it on host port 9003 as `vc-apigw`).

Route table (apigw, `internal/apigw/httpserver/service.go`):

| Method | Path | Purpose |
|---|---|---|
| GET | `/.well-known/openid-credential-issuer` | issuer metadata: `credential_endpoint`, `nonce_endpoint`, `credential_configurations_supported` keyed by **scope name** (`pid_1_8`, `mdl`, `diploma`, etc.) |
| GET | `/.well-known/oauth-authorization-server` | RFC 8414 AS metadata (`pushed_authorization_request_endpoint`, `token_endpoint`, ...) |
| GET | `/jwks`, `/iacas` | issuer signing keys / mDoc IACA trust anchors |
| POST | `/nonce` | one-time nonce for proof-of-possession JWTs |
| POST | `/op/par` | Pushed Authorization Request — **PKCE (`code_challenge`) is mandatory for public clients**; supports `authorization_details` array for scoped credential requests; supports wallet-attestation-based client auth via `OAuth-Client-Attestation`/`OAuth-Client-Attestation-PoP` headers as a fallback to static client_id |
| GET | `/authorize` | redirects into an internal consent flow (session cookie), eventually redirects back to the wallet's `redirect_uri` with an authorization code |
| POST | `/token` | `authorization_code` \| `urn:ietf:params:oauth:grant-type:pre-authorized_code` \| `refresh_token`. **Authorization-code flow requires DPoP.** Token type is always `"DPoP"`. Response includes `c_nonce`, optionally `authorization_details` with `credential_identifiers` |
| POST | `/credential` | **DPoP required on every call**, plus a proof JWT (`typ: openid4vci-proof+jwt`) bound to the one-time nonce. Body: `credential_configuration_id` XOR `credential_identifier` + `proofs.jwt[]` (batch). Response: `credentials: [...]` (base64url CBOR for mso_mdoc, compact SD-JWT string, or JSON-LD for VC 2.0), plus optional `transaction_id`/`c_nonce`/`notification_id` |
| POST | `/deferred_credential`, `/notification` | deferred issuance polling / issuance-event notification |
| GET | `/credential-offer/:credential_offer_uuid` | credential-offer resolution (for QR/deep-link based offers) |

**PID/mDL/Diploma bootstrapping** (all under `vc/bootstrapping/*.json` and
`sirosid-dev/fixtures/vc-bootstrapping/*.json`, mounted into vc-apigw):
`pid_1_8`/`diploma`/`ehic` are SD-JWT VC (`dc+sd-jwt`, VCTM-driven); `mdl` is
`mso_mdoc` (ISO 18013-5, doctype-driven, no VCTM needed — adding a new mdoc
doctype is pure config). Test users (`alice-001`/`bob-002`/`carol-003`) are
resolved via `identity_mappings.json` against `mini-oidc` OIDC login claims
(`given_name`/`family_name`). Existing OAuth clients in the dev config:
`e2e-test-client`/`e2e-test-client-2`, both public, both scoped to
`pid, pid_1_5, pid_1_8, diploma, ehic, mdl`. **A new client (YubiId) needs its
own client_id entry hand-added to `fixtures/vc-config.yaml`'s
`apigw.delivery.openid4vci.clients`** with its own custom-scheme
`redirect_uri` (mirroring the existing `siros-sample://callback` pattern) —
there is no self-service/dynamic client registration for OpenID4VCI purposes.

## 3. OpenID4VP credential presentation flow

Two distinct surfaces exist:

**(A) vc-verifier standalone RP-initiated flow** — the one a wallet actually
presents credentials to as a verifier: `GET /authorize` (RP requests a DCQL
query built from `scope`) → returns `deep_link_url`/`qr_code_data` of the form
`openid4vp://cb?client_id=...&request_uri=https://verifier/.../request-object/:session_id`
→ wallet fetches the signed JWT request object → wallet `POST
/verification/oidc-direct_post` with `vp_token` + `presentation_submission` →
RP polls `/poll/:session_id` and exchanges the resulting code at `/token` for
an OIDC id_token.

**(B) apigw's internal "OpenID4VP-as-authentication"** — used *during* VCI
issuance to identity-proof a user (e.g., present an existing PID to bootstrap
issuance of a new credential), response mode `direct_post.jwt` (JWE-encrypted).

**Only DCQL is supported, not Presentation Exchange** — `PresentationDefinition`
is explicitly commented out of the request object type. `vp_token` is
`map[string][]string` keyed by DCQL query id (single-scope wallets can send a
bare string under a `"_default"` key). `transaction_data` (OpenID4VP §8.4) is
supported end-to-end and hashed into the SD-JWT KB-JWT. Sample DCQL templates
for `pid`/`ehic`/`eduid` exist under `vc/presentation_requests/*.yaml`.
**Note: `mdl` is not currently listed as verifier-supported** in the dev
`vc-config.yaml` — presenting an mDL isn't wired on the verifier side yet in
this environment.

## 4. Trust PDP (go-trust/AuthZEN) integration

go-trust is consumed as an external Go module + HTTP service by both
go-wallet-backend and vc — **it is not itself deployed as a container by
either of those repos' own compose files**; sirosid-dev is what actually runs
go-trust instances (`go-trust-allow`/`-whitelist`/`-deny`) and wires the PDP
URL into both. **Empty `pdp_url`/`WALLET_TRUST_PDP_URL` = allow-all; non-empty
= default-deny, fail-closed on PDP unreachability.** This should be
transparent to a wallet client — it never talks to go-trust directly.

Key touchpoints:
- **go-wallet-backend**: `pkg/trust/service.go` fails closed if no PDP configured (explicit "no local trust bypass"); `internal/engine/oid4vci.go:903-967` blocks issuance from an untrusted issuer before completing the flow; `internal/engine/oid4vp.go:409-462` resolves/verifies presentation-request signer trust (DID or x509_san_dns schemes) before accepting a presentation request; `internal/api/authzen_proxy.go` exposes the client-facing `/v1/evaluate`+`/v1/resolve` proxy.
- **vc (apigw/verifier)**: `pkg/trust/gotrust.go`, `pkg/trust/jwt_verifier.go` — `EvaluateIssuerTrust` runs **before accepting any wallet-presented credential** in both the standalone verifier and apigw's OID4VP-as-auth path (i.e., before a PID presented during issuance is trusted). One implementation wrinkle: PDP-deny errors are plain `fmt.Errorf`, not typed API errors, so they currently surface as **HTTP 500** rather than 401/403 — worth knowing if a client needs to distinguish "PDP denied" from "server broke."
- **Not wired anywhere** in either repo: vc-issuer, vc-registry, and vc-wallet have zero go-trust integration; a second DID-resolution-oriented go-trust adapter (`pkg/keyresolver/authzen.go`) exists in vc as library code with no call sites yet.
- Dev config in both `sirosid-dev` and the raw `vc`/`go-wallet-backend` repos runs **allow-all by default** (no `pdp_url` set) unless `PDP=whitelist`/`PDP=deny` overlays are used.

## 5. R2PS / WSCD remote key provisioning

**Not implemented client-side in any of the four repos analyzed.** The R2PS
*server* (`go-r2ps-service`) is itself a separate sibling repo not cloned for
this analysis — `docker-compose.r2ps.yml`'s `r2ps-server` service builds from
`../go-r2ps-service`. The only real client of the protocol is the native
mobile stack: `siros-wscd-manager` (a Rust library, `r2ps` Cargo feature),
consumed by the `siros-sdk-kotlin` Android sample app — also not one of the
repos in scope here.

Protocol (per sirosid-dev's docs, not verified against source since the
client isn't in these repos): (1) OPAQUE-based registration tied to
`client_id`+`context`; (2) OPAQUE authentication → session ID + symmetric key;
(3) `P256Generate` request over the authenticated session → server generates
an EC P-256 keypair in a SoftHSM2-backed HSM; (4) signing requests by key ID
over the same session. Read-only admin/monitoring API exists at
`GET /admin/store/keys[?client_id=]`, `/admin/store/statuses/{ka,wia}` (port
8444) — **keys are never provisioned via this admin API**, only inspected.

`WALLET_R2PS_URL` is set on go-wallet-backend by sirosid-dev's compose
overlay but **is not referenced anywhere in go-wallet-backend's Go source** —
appears to be forward-looking/unused wiring today, not an active integration.

**Implication for YubiId integration**: if hardware-backed/remote signing is
in scope, it is an entirely separate, out-of-band concern from the OpenID4VCI/VP
HTTP flows above, requiring its own OPAQUE client implementation — there is no
existing Dart/Flutter or generic HTTP client to build from in any of the
analyzed repos.

## 6. Bootstrapping a brand-new wallet client (e.g. YubiId)

Required steps, none of which are self-service/dynamic:

1. **Discover issuer metadata** from vc-apigw's `.well-known/openid-credential-issuer`.
2. **Get a client_id added** to `fixtures/vc-config.yaml`'s `apigw.delivery.openid4vci.clients` (static YAML edit + apigw restart) — no dynamic client registration endpoint exists for OpenID4VCI purposes. (Contrast: vc-verifier *does* support dynamic RP registration via `POST /register`, RFC 7591/7592 — that's for verifiers, not wallets.)
3. **Run the OAuth2 PAR + PKCE + authorization_code (or pre-authorized_code) flow** against vc-apigw as described in §2, with DPoP on the token and credential requests.
4. **Separately register a WebAuthn passkey** directly with go-wallet-backend (`/user/register-webauthn-begin`/`-finish` or the newer `/auth/passkey/register/*` under the AS module) — this is unrelated to the OpenID4VCI session.
5. **If Android**: the app's signing-cert fingerprint must be added to *both* `assetlinks.json` (Digital Asset Links, served by whatever hosts the wallet frontend) *and* go-wallet-backend's `WALLET_SERVER_RP_ORIGINS`/`rp_origins` config as an `android:apk-key-hash:...` entry — doing only one passes one check but fails the actual WebAuthn ceremony. There is no dynamic API for this; it's static config (env var or YAML) requiring a restart. **No iOS equivalent (Associated Domains / apple-app-site-association) exists anywhere in this stack** — that's undocumented territory for a future iOS build.
6. **Auth/session model choice**: go-wallet-backend supports two coexisting models — legacy bearer-JWT-per-request (HS256, minted at WebAuthn login, no separate token endpoint), or the newer AS module (`cfg.AS.Enabled`) with session-cookie-based passkey/OIDC login + `POST /auth/token` exchange for short-lived asymmetric JWTs, published JWKS, and proper `DELETE /auth/session` logout. **The new AS model is the more complete/modern path** and is what a new client integration should probably target, though it's explicitly still "in progress" per `docs/new-as.md`.
7. **R2PS/WSCD key provisioning**, if used, is a fully separate OPAQUE handshake directly with the R2PS server (§5) — not part of the OAuth session at all.

## What could NOT be determined from these four repos

- **go-r2ps-service** (R2PS server implementation) and **siros-wscd-manager**
  (Rust WSCD client) — not cloned; R2PS is documented only at the protocol
  level in sirosid-dev's README/docs, not verified against source.
- **wallet-frontend** — not cloned; the existing reference OpenID4VCI/VP
  *client* implementation closest to what YubiId needs to replicate is
  actually `vc`'s own `internal/wallet/apiv1/{vci,vp}.go` (a Go CLI test
  client), not wallet-frontend.
- Whether `docs/ARCHITECTURE.md`/`API.md`/`COMPATIBILITY.md`/`DEPLOYMENT.md`/
  `MIGRATION.md` in go-wallet-backend contain anything materially different
  from the code-derived facts above — not deeply read (time-boxed); worth a
  follow-up pass if Phase 3 implementation needs deployment-config detail.
