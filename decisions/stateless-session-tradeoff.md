# Architectural Decision Record: Adopting a Stateless Session Architecture and Token Revocation Strategy

## Status

**Accepted**

---

## Context

A core architectural objective for the **Mini AuthService** is to achieve high performance, rapid horizontal scaling, and minimal dependency overhead. As described in [[summaries/architectural-overview]], we designed the microservice to be stateless, avoiding the need to replicate or look up session records in a central database for every inbound HTTP request. 

Instead, we employ [[concepts/stateless-sessions]] where identity context (e.g., username, role) is derived cryptographically from the JSON Web Token (JWT) payload claims, using `HS256` symmetric signing as described in [[concepts/jwt-security]] and decided in [[decisions/symmetric-vs-asymmetric-signing]].

However, this stateless model introduces a fundamental trade-off regarding **Token Revocation**:
* **The Problem**: Once a JWT is issued to a client, it is cryptographically valid until its expiration timestamp (`exp` claim) is reached. If a user logs out, or if a token is compromised, there is no native way in a purely stateless setup to invalidate that token immediately.
* **The Consequences**: If an attacker gains access to an active token, they can access protected endpoints via [[concepts/authentication-flows]] and the dependency injection guards in [[concepts/fastapi-dependency-injection]] until the token naturally expires.

### Evaluated Options

To address this challenge, we evaluated three design options:

1. **Purely Stateless (No Revocation mechanism)**
   * *Description*: Rely entirely on short token lifetimes (e.g., 15 minutes) as implemented in [[entities/src-auth]]. Once issued, tokens cannot be revoked.
   * *Pros*: Maximum performance. No database or cache reads during API validation. Extremely simple.
   * *Cons*: High security risk. No way to force-logout a compromised session.

2. **Distributed Denylist (e.g., Redis-based Token Invalidation)**
   * *Description*: Maintain an in-memory database (Redis) storing revoked token IDs (`jti` claims) or signatures. During token validation in [[concepts/fastapi-dependency-injection]], check the cache.
   * *Pros*: Near-instantaneous token revocation (within milliseconds).
   * *Cons*: Reintroduces state to the microservice. Introduces an external network hop (Redis latency) to every API validation check, partially defeating the performance advantage of stateless JWTs.

3. **Hybrid Architecture (Stateless Access Tokens + Stateful/Rotated Refresh Tokens)**
   * *Description*: Keep access token verification completely stateless and short-lived (e.g., 15 minutes). Couple this with a stateful refresh token cycle. Refresh tokens are stored securely in a database (see [[decisions/database-credential-storage]]) and validated during refresh requests.
   * *Pros*: Keeps the main API delivery layer in [[entities/src-main]] fast and database-free for resource requests. Security risks are mitigated because access tokens expire quickly, and re-issuance (refreshing) is gated by a stateful, revocable check.
   * *Cons*: A compromised access token remains valid for its brief lifetime (up to 15 minutes).

---

## Decision

We have decided to adopt the **Hybrid Architecture (Option 3)**.

1. **Stateless Resource Validation**:
   * API resource requests containing access tokens will be validated entirely in-memory using cryptographic signature checks inside the `verify_token` helper in [[entities/src-auth]]. No external database or cache queries will occur during standard authorization checks in [[concepts/fastapi-dependency-injection]].
   
2. **Stateful Lifetime Management**:
   * We will control session lifetimes by deploying secure, `HTTPOnly` cookie-based **Refresh Tokens** with rotation capabilities, as specified in [[decisions/token-lifecycle-refresh]].
   
3. **Revocation Boundaries**:
   * Token revocation (e.g., explicit logout, user password reset, session termination) will target the stateful *Refresh Token* layer.
   * When a user logs out, their active Refresh Token is deleted from the PostgreSQL storage (connected via [[entities/src-config]]). While their current Access Token will remain cryptographically valid until its natural 15-minute expiration, they will be blocked from obtaining any new Access Tokens.

This architecture strikes an optimal balance between maximum scalability for the general resource endpoints and immediate control over long-lived session lifecycle boundaries.

---

## Consequences

* **Scale & Performance**: The FastAPI router in [[entities/src-main]] can process high-throughput authorization requests with sub-millisecond validation latency, as no database reads are required to confirm token validity.
* **Window of Vulnerability**: If an access token is compromised, there is a maximum 15-minute window during which the token can still be used. We accept this trade-off for the performance benefits, but require that any highly sensitive systems requiring instantaneous zero-trust revocation implement a Redis-based cache lookup wrapper inside the `verify_token` dependency.
* **Infrastructure Footprint**: The microservice remains light, requiring a connection to PostgreSQL (established in [[decisions/database-credential-storage]] and [[entities/src-config]]) only during initial authentication (`/login`) and session extension (`/refresh`), keeping resource consumption predictable and cost-effective.