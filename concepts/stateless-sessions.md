# Stateless Sessions & Cryptographic Identity Verification

The **Mini AuthService** utilizes a **stateless session model** to handle user authorization and identity verification. Unlike traditional stateful session architectures, which store active sessions in a server-side database or key-value store (e.g., Redis) and map them to a client-side session ID cookie, a stateless architecture delegates session storage entirely to the client.

By packaging user identity data, authorization privileges, and metadata directly into a cryptographically signed payload—the JSON Web Token (JWT)—the microservice can instantly verify user identities at the API gateway or route level without executing a database query.

This document describes the architectural principles of stateless identity verification, how the identity context is derived from cryptographic claims, and the trade-offs of this model.

---

## The Core Principle of Stateless Identity

In a stateless session model, the server issues a self-contained token containing all the claims required to establish the user's identity context. 

```
+------------+               1. POST /login                +-------------+
|            | ------------------------------------------> |             |
|            | <------------------------------------------ |             |
|   Client   |          2. HTTP 200 (Signed JWT)           |    Mini     |
|            |                                             | AuthService |
|            |               3. GET /me                    |             |
|            | --- (Header: Authorization: Bearer <JWT>) -> |             |
|            | <------------------------------------------ |             |
+------------+              4. HTTP 200 (Profile)          +-------------+
```

1. **Authentication**: The client logs in by posting credentials to `/api/v1/auth/login` (see [[concepts/authentication-flows]]).
2. **Token Generation**: The server validates the credentials and constructs an arbitrary dictionary of claims (such as the subject, `sub`). It signs this dictionary using a secret key to produce a JWT (see [[entities/src-auth]]).
3. **Transmission**: The token is sent to the client, which stores it (e.g., in memory, `localStorage`, or cookies).
4. **Verification**: For subsequent protected requests, the client attaches the JWT to the `Authorization` header. The server verifies the cryptographic signature of the token. If the signature is valid and the token has not expired, the server trusts the claims inside the token implicitly. **No database lookups are performed to retrieve session state.**

---

## Cryptographic Derivation of Identity Context

Identity context is not looked up; it is **derived** directly from the validated token payload. This is made possible by standard JWT claims, which provide assertions about the subject of the token.

### Standard Claims Mapping

The codebase leverages standard JSON Web Token claims to represent identity:

* **`sub` (Subject)**: Identifies the principal that is the subject of the JWT. In this service, the `sub` claim contains the unique username of the authenticated user.
* **`exp` (Expiration Time)**: Identifies the expiration time on or after which the JWT must not be accepted for processing. This is calculated dynamically during token creation:
  
$$\text{Expiration Time} = \text{Current UTC Time} + \text{Access Token Lifetime}$$

### Code-Level Context Generation

When a user authenticates, the system serializes the identity context into claims using the core cryptographic engine in `src/auth.py` (see [[entities/src-auth]]):

```python
def create_access_token(data: dict, expires_delta: timedelta = None):
    to_encode = data.copy()
    expire = datetime.utcnow() + (expires_delta or timedelta(minutes=15))
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, settings.JWT_SECRET, algorithm=settings.ALGORITHM)
```

The data map parsed by the router at login contains:
```json
{
  "sub": "admin"
}
```

When decoded, this payload is validated against the signature computed using the symmetric `HS256` key stored in the application's configuration (see [[concepts/jwt-security]] and [[concepts/configuration-management]]). If valid, the resulting dictionary is injected directly into the route handlers.

---

## The Identity Extraction Pipeline

Integrating stateless sessions cleanly into a FastAPI application is achieved using **Dependency Injection** (see [[concepts/fastapi-dependency-injection]]). Rather than manually parsing headers and validating cryptographic signatures within every route handler, the delivery layer leverages declarative dependencies.

### Step-by-Step Flow

The processing lifecycle of an incoming request to a protected endpoint (e.g., `/api/v1/auth/me`) proceeds as follows:

```
 Incoming HTTP Request
        │
        ▼
 ┌────────────────────────────────────────┐
 │  FastAPI Router Intercepts Request    │ (src/main.py)
 └──────────────────┬─────────────────────┘
                    │
                    ▼  Invokes Dependency
 ┌────────────────────────────────────────┐
 │   verify_token(token: str) Dependency  │ (src/auth.py)
 └──────────────────┬─────────────────────┘
                    │
       ┌────────────┴────────────┐
       ▼                         ▼
 [Signature Valid]        [Signature Invalid / Expired]
       │                         │
       │ Decode Claims           │ Raise HTTP 401 Unauthorized
       ▼                         ▼
 ┌──────────────────┐      ┌────────────────────────────┐
 │ Return Dict to   │      │ Request Rejected           │
 │ Route Handler    │      └────────────────────────────┘
 └─────────┬────────┘
           │
           ▼  Injected as `token_data`
 ┌────────────────────────────────────────┐
 │ Endpoint: Return {"username": sub}     │ (src/main.py)
 └────────────────────────────────────────┘
```

This pipeline is represented in the codebase within `src/main.py` (see [[entities/src-main]]):

```python
@app.get("/api/v1/auth/me")
async def get_current_user(token_data: dict = Depends(verify_token)):
    if not token_data:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid or expired token",
        )
    return {"username": token_data["sub"], "status": "active"}
```

---

## Architectural Benefits and Trade-offs

Choosing a completely stateless model significantly impacts the security posture, operational efficiency, and scalability of the microservice.

### Benefits

1. **Massive Horizontal Scalability**: Because the application server does not persist sessions, instances of the Mini AuthService can be duplicated endlessly behind a round-robin load balancer. There is no session state to synchronize, and no centralized database is queried on incoming authorized requests.
2. **Sub-millisecond Validation**: Checking an HMAC signature (`HS256`) is extremely fast. It requires no network or disk I/O, minimizing response latency for protected API routes.
3. **Decoupled Architecture**: Any service sharing the symmetric key (`JWT_SECRET`) and configuration algorithm can locally verify tokens without calling the Mini AuthService directly. See [[decisions/symmetric-vs-asymmetric-signing]] for trade-offs regarding this secret sharing model.

### Trade-offs & Mitigations

| Trade-off / Risk | Impact | Architectural Mitigation |
| :--- | :--- | :--- |
| **Difficult Revocation** | Once issued, a token is valid until its `exp` claim passes. Logouts or immediate user bans are not instantly recognized. | Keep access token lifecycles short (e.g., 15 minutes) and issue a secure, statefully-tracked Refresh Token via HTTPOnly cookies (see [[decisions/token-lifecycle-refresh]] and [[decisions/stateless-session-tradeoff]]). |
| **Database Synchronization** | If user permissions, passwords, or active status change in the database (see [[decisions/database-credential-storage]]), the active token remains valid with stale metadata. | Force a token rotation scheme, check critical parameters during token refresh cycles, or use a hybrid approach (e.g., lightweight Redis caching for blacklisting revoked tokens). |
| **Payload Bloat** | Since all session context must fit inside the token, adding too many claims (e.g., full profile data, deep permission arrays) increases the size of every HTTP request. | Store only core identifiers (`sub`, `roles`) inside the payload claims. Use downstream microservices to fetch auxiliary user profiles based on the verified `sub` context. |

---

## Related Guides and Decisions
* For detail on how standard JWT claims are constructed and signed, refer to [[concepts/jwt-security]].
* For details on the dependency injection mechanism that wraps this pipeline, see [[concepts/fastapi-dependency-injection]].
* To review the security trade-offs of using symmetric key schemes for stateless tokens, read [[decisions/symmetric-vs-asymmetric-signing]].
* For the design decisions around token expiration, revocation, and lifetimes, see [[decisions/stateless-session-tradeoff]] and [[decisions/token-lifecycle-refresh]].