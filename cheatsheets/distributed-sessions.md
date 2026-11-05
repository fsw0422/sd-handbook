# JWT and distributed sessions cheatsheet

**The main choice is where each request gets its authorization information: current server state or a previously issued snapshot.** Shared sessions, protected cookies, and JWT access tokens make different tradeoffs between remote lookups and how quickly access changes take effect.

## Key terms

| Term | Meaning |
| --- | --- |
| Authentication | Establish who the caller is. |
| Authorization | Decide whether the caller may perform an action on a resource. |
| Login session | The application's record or representation of an ongoing login. |
| Cookie | A browser mechanism for storing and sending a value. It can carry an opaque session ID or a token. |
| JWT | A format for claims. It can represent an access token, an ID token, or session data. |
| Access token | A credential presented to an API. It can be a JWT or an opaque value. |

A cookie, a JWT, and a session describe different parts of the design. Choosing JWTs does not automatically make the whole system stateless.

## What a JWT looks like

**A JWT is three base64url strings joined by dots: `header.payload.signature`.** This classic example token is signed with HS256:

```text
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

Split at the dots:

```text
header     eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
payload    eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ
signature  SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

| Part | Decoded | What it does |
| --- | --- | --- |
| Header | `{"alg":"HS256","typ":"JWT"}` | Names the signing algorithm. |
| Payload | `{"sub":"1234567890","name":"John Doe","iat":1516239022}` | Holds the claims: who the token is about, and when it was issued. |
| Signature | 32 raw bytes | HMAC-SHA256 over `header.payload`, here keyed with the example secret `your-256-bit-secret`. |

The issuer builds it like this:

```text
part1 = base64url(header JSON)
part2 = base64url(payload JSON)
part3 = base64url(sign(key, part1 + "." + part2))

token = part1 + "." + part2 + "." + part3
```

A verifier recomputes the signature over the first two parts, so changing one character of the payload breaks it.

A real access token or ID token has the same shape, only longer. Its header usually names an asymmetric algorithm and a key ID, as shown in [JWT, JWS, and JWE](#jwt-jws-and-jwe). Its payload carries claims such as `iss`, `aud`, and `exp`, as shown in [authorization claims in JWTs](#authorization-claims-in-jwts). The signature's length depends on the algorithm:

| Algorithm | Signature | Base64url characters |
| --- | --- | --- |
| HS256 | 32 bytes | 43 |
| ES256 | 64 bytes | 86 |
| RS256 with a 2048-bit key | 256 bytes | 342 |

- **Almost every JWT starts with `eyJ`.** That is `{"` in base64url, so JWTs are easy to spot in logs and headers.
- **Anyone can decode it without a key.** The signature stops changes but hides nothing, so keep secrets out of the payload.
- **It usually travels as a bearer token:** `Authorization: Bearer eyJ...`.
- **An encrypted JWT (JWE) has five parts** instead of three, as the next section shows.
- **Don't paste real tokens into online decoders.** A live token works as a credential until it expires.

## JWT, JWS, and JWE

**"JWT" almost always means a signed JWS token, so anyone holding it can read the payload.** The JWT specification defines only the claims set. JWS and JWE define how those claims are protected. A JWS header and payload are only base64url-encoded JSON, so the token's security depends on checking the signature.

| Form | Protects | Compact shape | Spec |
| --- | --- | --- | --- |
| JWS | Integrity. Any change breaks the signature. | `header.payload.signature` (3 parts) | RFC 7515 |
| JWE | Confidentiality and tamper detection | `header.encryptedKey.iv.ciphertext.tag` (5 parts) | RFC 7516 |
| Unsecured JWT | Nothing. `alg` is `none` and the signature is empty. | `header.payload.` | RFC 7519, section 6 |

Decoded header of a JWS access token:

```json
{ "alg": "RS256", "typ": "at+jwt", "kid": "2026-10-key-1" }
```

`alg` names the signing algorithm, `kid` selects a key from the issuer's published key set, and `typ` marks the token as an access token. The API checks `typ` and accepts only algorithms it has configured. It never lets the token header choose. JWT libraries must let the caller fix the allowed algorithms, refuse all others, and use each key with exactly one algorithm. This blocks a forged header that switches `alg` to `none`, or from RS256 to HS256 so the RSA public key is misused as an HMAC secret. Explicit typing is recommended for new kinds of JWT, so a token issued for one purpose cannot pass as another.

### What JWS gives you

**A signed token lets each API check its claims locally. With an asymmetric algorithm, being able to verify tokens does not give the power to create them.**

| Benefit | How it helps |
| --- | --- |
| Tamper evidence | Changing any header or payload byte breaks the signature. A client cannot widen its own `scope` or extend `exp`. |
| Issuer proof | With RS256 or ES256, only the issuer's private key can sign. Every API verifies with the public key, so a compromised API still cannot mint tokens. |
| Local validation | APIs verify with cached public keys instead of calling the issuer on every request. This is the saving described in [where session lookups happen](#where-session-lookups-happen). |
| Readable claims | Anyone holding the token can read the claims without a key. A client can read its ID token, but it must still treat access tokens as opaque. The JWT access token profile forbids clients from inspecting them, because the issuer can change the format at any time. If claims are sensitive, the issuer can encrypt the token, leave those claims out, or use an opaque token. |

Base64url is an encoding, not encryption. Keep secrets and unneeded personal data out of JWS payloads.

HMAC algorithms such as HS256 sign and verify with one shared secret, so every verifier can also create tokens. They fit a service that issues and checks its own tokens. The key must be at least as long as the hash output, so at least 256 bits for HS256. Verifiers must compare the MAC in constant time to avoid timing attacks. [signatures and public-key encryption](tls-docsign.md#symmetric-and-asymmetric-cryptography)

### What JWE gives you

**An encrypted token hides its claims from everyone except the recipient, including the parties that store and forward it.**

| Benefit | How it helps |
| --- | --- |
| Confidentiality | Only the holder of the decryption key can read the claims. The browser, proxies, and logs see ciphertext. |
| Protection after TLS | TLS protects one connection. Tokens also sit in browser storage, log lines, and later hops, where JWE still hides the claims. See [where TLS protection ends](tls-docsign.md#where-tls-protection-ends). |
| Recipient targeting | A token encrypted to one API's public key is readable only by that API. `aud` asks other services to reject a token, while encryption stops them from reading it. |
| Tamper detection | JWE requires authenticated encryption, so a modified token fails to decrypt. The required `enc` header must name an AEAD algorithm with a specified key length, such as `A256GCM`. If the authentication tag does not match, the recipient rejects the token without releasing any plaintext. |

These benefits cost key management and visibility. Each recipient must protect a decryption key, tokens grow larger, and debugging needs that key. **Decryption alone does not prove who created the token.** With public-key encryption, anyone holding the recipient's public key can create one.

### Choosing a form

- **Signed JWS:** the default for access tokens and ID tokens. TLS protects them in transit, and APIs mainly need integrity and local verification.
- **Encrypted JWE:** when the token passes through a party that should not read its claims, such as a browser holding a token with personal data.
- **Nested JWT:** when the recipient needs both secrecy and proof of the issuer. Sign first, then encrypt the signed token. The JWT spec allows either order, but signing first is the normal choice. The encryption then hides the signature and the signer, and an attacker cannot strip the signature off. Because JWE always uses authenticated encryption, there is no need to sign again after encrypting.
- **Opaque token:** when only the issuer should read the claims. Introspection avoids encryption keys at every API, at the cost of a remote check. See [where session lookups happen](#where-session-lookups-happen).

Example: an access token for `orders-api` carries the user's email and passes through the browser. As a JWS, the browser can read the email but cannot change `scope`. As a nested JWT, the browser sees only ciphertext:

```mermaid
sequenceDiagram
    participant Issuer
    participant Browser
    participant API as orders-api

    Issuer->>Issuer: Sign claims with issuer private key
    Issuer->>Issuer: Encrypt signed token with orders-api public key
    Issuer-->>Browser: Nested JWT
    Note over Browser: Sees only ciphertext
    Browser->>API: Request with nested JWT
    API->>API: Decrypt with orders-api private key
    API->>API: Verify signature with issuer public key
```

## Authorization claims in JWTs

Selected claims from a decoded access token:

```json
{
  "iss": "https://auth.example.com",
  "sub": "user-123",
  "aud": "orders-api",
  "scope": "orders:read orders:write",
  "roles": ["admin"],
  "iat": 1793452800,
  "exp": 1793453400
}
```

`iss` identifies the issuer, `sub` the subject, and `aud` the intended recipient. `iat` and `exp` are Unix timestamps in seconds, here ten minutes apart. The JWT spec makes every registered claim optional and lets each application decide which ones it requires. If `aud` is present and does not name the recipient, the recipient must reject the token. A token must not be accepted at or after `exp`, though verifiers may allow a small leeway for clock skew, usually no more than a few minutes.

`scope` describes granted access. The JWT access token profile says the token should carry `scope` when the client requested scopes, and each scope value must have meaning for the resources named in `aud`. `roles` is a registered claim, but values such as `admin` have application-specific meanings. The profile recommends the names `roles`, `groups`, and `entitlements` for such attributes but defines no standard role values. The API combines these claims with its own access rules. For example, `orders:read` alone does not establish that the user may read another tenant's order.

For a signed JWT access token, the API must:

1. Verify the signature using trusted keys and an allowed algorithm.
2. Validate the expected token type, issuer, audience, expiration, and other required claims.
3. Check whether the caller may perform this action on this resource, including tenant boundaries and ownership where relevant.

**Decoding a JWT does not validate it. A valid JWT does not automatically authorize every action.** Validation rules must match the token's intended purpose. When one issuer creates several kinds of JWT, such as ID tokens and access tokens, the rules for each kind must reject the others, for example through `typ`, `aud`, or separate keys. If any signature or decryption step fails, including either layer of a nested JWT, the whole token must be rejected.

For JWT access tokens, the access token profile turns steps 1 and 2 into required checks:

1. `typ` is `at+jwt` or `application/at+jwt`. Any other value is rejected.
2. `iss` exactly matches the expected issuer.
3. `aud` contains the API's own identifier.
4. The signature verifies with a key provided by the authorization server. A token with `alg` set to `none` is rejected.
5. The current time is before `exp`, with an optional small leeway for clock skew.

Any failed check returns an `invalid_token` error.

## Where session lookups happen

| Layer | Representation | Remote check without caching | Revocation behavior |
| --- | --- | --- | --- |
| Browser session | Opaque ID referencing a shared record | Normally one lookup per authenticated request | Next authoritative lookup sees session revocation |
| Browser session | Session data inside a protected cookie | None for session validation | A copied cookie can work until expiry unless additional state invalidates it |
| API access | JWT validated locally | None for token validation | Existing token works until expiry unless another check rejects it |
| API access | Opaque token checked through introspection | Call the authorization server to check token status | Next current status check sees token revocation |

Protected cookies carry session data with integrity protection and optional encryption. Cookies therefore do not imply a shared store. For example, Django's signed-cookie session backend signs the data with the server's secret key but does not encrypt it, so the user can read it. The signature proves the data is unchanged, not that it is the latest copy. An older signed cookie still works, and logout does not invalidate it, until it is older than the configured session age. Browsers commonly cap a cookie at 4096 bytes, which limits how much session data fits.

Introspection asks the authorization server whether a token is active and what it represents. It can be used with JWTs too. Token format and validation strategy are separate choices. The API posts the token to the introspection endpoint and must authenticate, so outsiders cannot probe for valid tokens. The response always has an `active` flag and may add fields such as `scope`, `client_id`, `sub`, `aud`, `iss`, and `exp`. For an expired, revoked, or unknown token, or one this API may not use, the server must return `active: false` and should add nothing else.

These are session or token checks. Resource authorization may still require database reads. Verification keys can be cached locally, so JWT signature verification does not normally contact the issuer on every request.

Shared stores such as Redis let any application instance handle the same user's requests without sticky routing. **Stateless application instances can still depend on a shared session store.** Keeping sessions on individual servers needs sticky routing, which creates hot spots and breaks failover. A common Redis layout stores one hash per session ID, with a key expiry that resets on each request, so inactive sessions expire without cleanup jobs. A set of session IDs per user supports logout on all devices.

## Reducing shared-store reads

A local cache in each backend can avoid repeated session lookups. The same idea applies to caching introspection results. It adds a period during which the backend may use an outdated decision. The introspection standard allows APIs to cache responses but requires system designers to weigh this tradeoff. A short cache gives fresher answers at the cost of more calls. A longer cache cuts load, but a token revoked during the window can still be accepted. Very sensitive systems can turn caching off. A cached response must not be used past the `exp` it contains.

| Strategy | What happens after revocation |
| --- | --- |
| Read current authoritative state each time | Subsequent checks reject access once the change is visible |
| Cache for 30 seconds | An entry validated just before revocation can remain usable for almost 30 seconds |
| Push invalidation to backend caches | Rejection starts when each backend processes the event; missed events need recovery |
| Accept a JWT for five minutes | A token issued just before revocation can remain usable for almost five minutes |

These are illustrative durations. Cache expiry must be bounded from the last authoritative validation, capped by credential expiry, and not extended indefinitely on cache hits. Replication delay, clock tolerance, and outage behavior also affect the actual revocation delay.

## Hybrid browser session and API access

A Backend for Frontend, or BFF, manages OAuth tokens and calls APIs on the browser's behalf. Its session can use either a shared store or a protected cookie. The BFF pattern does not require JWT access tokens. Current IETF guidance ranks it highest for security among three browser OAuth architectures, while recognizing the added backend and proxy cost. From most to least secure:

1. **BFF:** a backend acts as a confidential OAuth client, keeps all tokens, and proxies every API call. Browser code never receives tokens.
2. **Token-mediating backend:** a confidential backend obtains tokens but hands the access token to the browser, which calls APIs directly.
3. **Browser-only OAuth client:** the browser runs OAuth as a public client and is exposed to every attack the guidance covers.

The BFF cost is a more complex system with a server that carries, and can see, all API traffic. With a shared store, the cookie holds only a session ID and the BFF can revoke sessions at any time. With a protected cookie that holds tokens, the BFF should encrypt it, and revoking the tokens cuts off access.

One implementation uses a shared session store and JWT access tokens:

```mermaid
sequenceDiagram
    participant Browser
    participant Backend as Any backend or BFF instance
    participant Store as Shared session store
    participant API as Downstream API

    Browser->>Backend: Request with opaque session cookie
    Backend->>Store: Look up session
    Store-->>Backend: Session lookup result
    alt Session is active
        Backend->>API: Request with JWT access token for this API
        API->>API: Validate token and authorize action
        API-->>Backend: Response
        Backend-->>Browser: Response
    else Session is missing, expired, or revoked
        Backend-->>Browser: Require authentication
    end
```

**This implementation reads the shared store for each authenticated browser request unless it caches sessions.** Downstream services validate the JWT locally. The diagram assumes a usable access token already exists. Use `Secure`, `HttpOnly` session cookies, suitable `SameSite` settings, and CSRF protection. The BFF guidance requires `Secure`, `HttpOnly`, and a CSRF defense. It recommends `SameSite=Strict`, `Path=/`, no `Domain` attribute, and a `__Host-Http-` cookie name prefix. `SameSite` alone stops CSRF only if no other application shares the BFF's site. Another option is to require a custom request header, which a cross-site page cannot send without passing a CORS preflight.

## Permission changes and logout

A JWT contains a snapshot of claims. **Roles copied into a shared session record can also become stale.** Reading Redis on every request only returns the stored record; a role change in the user database must update or invalidate that record, or authorization must read current permissions separately.

Treat these as separate operations:

| Operation | What it changes |
| --- | --- |
| Delete the browser cookie | Removes that browser's copy; a copied credential may still work |
| Revoke the application session | Stops requests at backends that check the updated session state |
| Revoke the refresh token or grant | Stops renewal through the revoked credential or grant |
| Stop use of existing access tokens | APIs must learn and enforce revocation, or access continues until token expiry |
| Remove a role or resource permission | Takes effect where authorization checks see the updated permission |

Revoking a refresh token does not by itself make an existing JWT fail local signature and expiry checks. Issuer-side revocation may also invalidate associated access tokens, but APIs need a way to observe that change. The token revocation standard requires servers to support revoking refresh tokens and recommends supporting access token revocation. When a server supports both, revoking a refresh token should also invalidate access tokens from the same grant. For self-contained tokens such as JWTs, the API learns of this only through a backend link to the authorization server, which the standard does not define. Opaque tokens that the API checks with the server can be cut off directly. Short token lifetimes limit how long a revoked token stays usable.

Example: a JWT issued at **12:00** expires at **12:05**. At **12:01**, the session and refresh grant are revoked. A backend checking that session blocks new requests, but an API presented with the existing JWT may accept it until 12:05 if it checks only the JWT. Renewed tokens must also reflect updated permissions for short lifetimes to bound how long removed permissions remain usable.

Coordinated logout can notify applications through OpenID Connect Back-Channel Logout. Each application still needs to invalidate its own session. The identity provider sends a signed logout token, which is a JWT, in a direct server-to-server POST to each application's registered logout URI. The browser is not involved, so the application cannot use cookies and must find the session from the token's claims. The token must include `iss`, `aud`, `iat`, `exp`, `jti`, an `events` claim marking a back-channel logout, and `sub`, `sid`, or both. It must not include `nonce`, so it cannot pass as an ID token. The application validates it like an ID token, ends the matching sessions, and returns 200, or 400 if validation fails. Without `sid`, all of that user's sessions from that issuer end. The application should also revoke the session's refresh tokens unless they were issued for offline access.

## Choosing a design

- **One browser app and its own backend:** A cookie session can be sufficient. Adding OAuth access tokens between those two components is optional. The browser-app guidance says OAuth between a frontend and backend on the same domain is often not needed. Server-side cookie session state can track the login, and OpenID Connect can still delegate sign-in to an identity provider. OAuth can still help when adding more domains or centralizing login and MFA.
- **Central session control:** An opaque ID and shared store give a direct place to revoke sessions. Local caching adds a freshness tradeoff.
- **Independent API validation:** Short-lived JWTs reduce remote token checks. Accept the remaining validity window or add revocation checks.
- **OAuth refresh tokens:** When issuing them to public clients, authorization servers must use rotation or cryptographic sender binding to detect replay. With rotation, each refresh returns a new refresh token and invalidates the old one. If an old token is used again, the server cannot tell the attacker from the real client, so it revokes the active token and the client must get a new grant. Sender binding ties the token to a client key through mutual TLS or DPoP. Servers should also expire refresh tokens after a period of client inactivity, and may revoke them on events such as a password change or logout.
- **Fast permission changes:** Decide exactly which component checks current permissions, how caches are invalidated, and what happens if that state is unavailable.
