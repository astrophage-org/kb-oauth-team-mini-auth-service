# Symmetric (HS256) vs. Asymmetric (RS256) Token Signing

## Status

**Accepted**

## Context

In designing the stateless token verification architecture for the **Mini AuthService** (detailed in [[summaries/architectural-overview]]), we must establish how identity tokens are cryptographically signed and verified. The primary goal of the authentication microservice is to issue secure JSON Web Tokens (JWT) that downstream clients and API routes can confidently trust without referencing centralized database state on every incoming request (see [[concepts/stateless-sessions]]).

The security and performance of this verification flow rest heavily on the chosen cryptographic signing scheme. Two industry-standard pathways exist for securing JWTs:

1. **Symmetric Signing (HS256)**: Uses a single secret key (`JWT_SECRET`) shared between the token issuer (the auth service) and any verifying party. The same secret key is used to both sign and verify the token signature.
2. **Asymmetric Signing (RS256 / ES256)**: Uses a public/private key pair. The token issuer retains a secure private key to sign the tokens, while downstream consumers or gateways use a publicly distributed public key to verify the signatures.

Our current implementation in [[entities/src-auth]] utilizes the symmetric **HS256** algorithm with a secret key retrieved from configuration settings (see [[entities/src-config]]). As we look to productionize our configuration practices (as discussed in [[concepts/configuration-management]]), we must formally evaluate whether symmetric signing satisfies our system's architecture, security boundaries, and scalability requirements, or if we should transition to an asymmetric alternative.

---

## Decision

We have decided to **retain the HS256 symmetric signing scheme** for token issuance and verification within the Mini AuthService ecosystem. 

This decision is motivated by the specific architectural boundaries of our system, key performance advantages, and operational simplicity.

```
+---------------------------------------------------------------------------------+
|                                 Trust Boundary                                  |
|                                                                                 |
|  +--------------------+         Signed Token (HS256)        +----------------+  |
|  |  Auth Core Engine  | ==================================> | FastAPI Router |  |
|  |  ([[entities/src-auth]])  |                                     | ([[entities/src-main]]) |  |
|  +----------+---------+                                     +--------+-------+  |
|             ^                                                        ^          |
|             | Uses                                                   | Uses     |
|             +-------------------> [ JWT_SECRET ] <-------------------+          |
|                                (Single Shared Secret)                           |
+---------------------------------------------------------------------------------+
```

### Rationale

1. **Colocated Issuance and Verification**:
   In our current architecture, the Mini AuthService functions as both the *issuer* of the tokens (via the `/api/v1/auth/login` endpoint) and the *verifier* (using dependency-injection guards like `verify_token` in [[concepts/fastapi-dependency-injection]]). Because the same physical service boundary handles both signing and validation, there is no immediate architectural need to share the key material outside our trust domain.

2. **Computational and Performance Efficiency**:
   HS256 utilizes symmetric HMAC-SHA256 computations, which require significantly fewer CPU cycles than RSA or ECDSA mathematical operations. Because the validation guards run inline within FastAPI request dependencies on endpoints like `/api/v1/auth/me` (see [[entities/src-main]]), choosing HS256 keeps cryptographic verification latency to a sub-millisecond overhead. This directly aligns with the microservice's goal of high throughput and rapid request validation.

3. **Reduced Configuration Complexity**:
   Implementing asymmetric schemes (like RS256) requires managing private/public key pairs, setting up key rotation routines, and exposing a JWKS (JSON Web Key Set) endpoint. By standardizing on HS256, our configuration footprint in [[concepts/configuration-management]] remains extremely lightweight, requiring only a single, highly entropy-dense string as the `JWT_SECRET`.

---

## Consequences & Trade-offs

While HS256 offers clear performance and simplicity advantages, it introduces specific trade-offs that we must mitigate:

* **Symmetric Key Compromise Risk**:
  If an attacker gains access to the `JWT_SECRET`, they can forge arbitrary tokens and bypass all authorization controls. We will mitigate this by upgrading the system to strict 12-factor compliance via `pydantic-settings` to ensure the secret is never checked into git, and instead is injected via environment variables at runtime (see [[concepts/configuration-management]]).
* **Decentralized Verification Limitations**:
  If the Mini AuthService scales into an ecosystem with third-party microservices that must verify tokens independently without calling the auth microservice, distributing the `JWT_SECRET` increases the attack surface. In such a scenario, we must either:
  * Route all external validation traffic through an API gateway that handles verification centrally.
  * Re-evaluate asymmetric schemes once external, untrusted consumers require verification capabilities.
* **Compensating Security Postures**:
  To limit the window of opportunity if a token or a secret is compromised, we will enforce tight lifetimes on JWT access tokens and implement refresh token mechanics (see [[decisions/token-lifecycle-refresh]] and [[decisions/stateless-session-tradeoff]]).

## Related Topics
* Cryptographic application of the selected algorithm: [[concepts/jwt-security]]
* Sequential implementation of the login/verification flows: [[concepts/authentication-flows]]
* Centralized handling of configuration variables: [[concepts/configuration-management]]