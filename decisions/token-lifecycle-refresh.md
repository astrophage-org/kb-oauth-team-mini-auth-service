# Architectural Decision Record: Token Lifecycles and Secure HTTPOnly Refresh Tokens

## Status

**Accepted**

---

## Context

In our initial security model, the **Mini AuthService** implemented standard stateless OAuth2 user authentication where a client exchanged credentials via `POST /api/v1/auth/login` for a cryptographically signed JSON Web Token (JWT) using the symmetric `HS256` algorithm (see [[concepts/jwt-security]] and [[decisions/symmetric-vs-asymmetric-signing]]). 

However, this implementation suffered from several architectural deficiencies:
1. **Short-Lived Access Tokens Only**: The access tokens were configured with a hardcoded expiration of 15 minutes. Once expired, clients had to prompt users to re-enter their credentials. This represents a highly disruptive user experience.
2. **Insecure Storage Temptation**: To avoid forcing users to log in every 15 minutes, client application developers are frequently tempted to request excessively long token lifetimes (e.g., 30 days) and store these access tokens in insecure browser environments like `localStorage` or `sessionStorage`. This exposes the system to catastrophic token theft via Cross-Site Scripting (XSS) attacks.
3. **The Revocation Dilemma**: Because our session architecture is stateless (as evaluated in [[concepts/stateless-sessions]] and [[decisions/stateless-session-tradeoff]]), verifying an access token requires no database lookups. This design makes access tokens impossible to invalidate or revoke instantly before their natural expiration without violating statelessness.

To resolve these challenges, we need to partition token lifecycles into a **dual-token model**:
* **Access Tokens**: Short-lived, stored in-memory by the client, and used for rapid, stateless API authorization.
* **Refresh Tokens**: Longer-lived, stored securely in the client browser using specialized HTTP cookies, and used solely to acquire fresh access tokens.

---

## Decision

We will implement a secure, dual-token lifecycle policy utilizing **HTTPOnly Cookies** for transmission of refresh tokens. This architecture maintains the performance benefits of stateless API calls while establishing a highly secure mechanism for silent session renewal.

```
+----------------+                +-------------------------+                +-------------------------+
|  HTTP Client   |                |   FastAPI Router Layer  |                |  Authentication Engine  |
|                |                |    ([[entities/src-main]])     |                |    ([[entities/src-auth]])     |
+-------+--------+                +------------+------------+                +------------+------------+
        |                                      |                                      |
        |  1. POST /login                      |                                      |
        |------------------------------------->|  2. Authenticate                     |
        |                                      |------------------------------------->|
        |                                      |  3. Generate both tokens             |
        |                                      |<-------------------------------------|
        |  4. HTTP 200 (JSON access_token)     |                                      |
        |     + Set-Cookie: refresh_token      |                                      |
        |<-------------------------------------|                                      |
        |                                      |                                      |
        |  5. GET /me (Bearer Header)          |                                      |
        |------------------------------------->|  6. Validate Access Token            |
        |                                      |------------------------------------->|
        |  7. HTTP 200 (User Context)          |<-------------------------------------|
        |<-------------------------------------|                                      |
        |                                      |                                      |
        |  [ Access Token Expires ]            |                                      |
        |                                      |                                      |
        |  8. POST /refresh (Cookie Included)  |                                      |
        |------------------------------------->|  9. Validate Refresh Token           |
        |                                      |------------------------------------->|
        |  10. HTTP 200 (New access_token)     |                                      |
        |<-------------------------------------|                                      |
```

### 1. Token Specification and Lifetimes

| Token Type | Lifetime | Delivery Mechanism | Storage Location | Intended Usage |
| :--- | :--- | :--- | :--- | :--- |
| **Access Token** | 15 Minutes | JSON Response Body | Client Memory (JavaScript State) | Included as a Bearer Token in the `Authorization` header for API resource authorization. |
| **Refresh Token** | 7 Days | `Set-Cookie` Header | Secure Browser Sandbox (No JS access) | Sent automatically by the browser to `/api/v1/auth/refresh` to obtain new access tokens. |

### 2. Secure HTTPOnly Cookie Attributes
To completely immunize the refresh token against Cross-Site Scripting (XSS) and mitigate Cross-Site Request Forgery (CSRF) risks, the `Set-Cookie` header generated by [[entities/src-main]] during authentication must declare the following strict browser directives:

```http
Set-Cookie: refresh_token=<token_value>; Max-Age=604800; Path=/api/v1/auth/refresh; HttpOnly; Secure; SameSite=Strict
```

* **`HttpOnly`**: Absolutely blocks client-side JavaScript from accessing the cookie payload via `document.cookie`. This mitigates the risk of token theft via XSS.
* **`Secure`**: Restricts the cookie transmission strictly to encrypted `HTTPS` connections. (This policy can be relaxed automatically on `localhost` during local development, driven by dynamic configuration toggles).
* **`SameSite=Strict`**: Instructs the browser to suppress sending this cookie during cross-site requests, providing robust protection against CSRF attacks.
* **Scope Restriction (`Path=/api/v1/auth/refresh`)**: Restricts cookie transmission *only* to requests targetting the specific token-refresh path. The browser will not append the refresh token cookie on standard API calls to protected endpoints (e.g., `/api/v1/auth/me`), minimizing exposure of the high-value refresh credential.

### 3. Core Engine Upgrades

#### Configuration Management ([[entities/src-config]])
The Configuration Manager must be upgraded in accordance with [[concepts/configuration-management]] to support these parameters dynamically via Pydantic Settings:

```python
# src/config.py additions
class Settings(BaseSettings):
    # ... existing settings ...
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 15
    REFRESH_TOKEN_EXPIRE_DAYS: int = 7
    COOKIE_SECURE: bool = True
    COOKIE_SAMESITE: str = "strict"
```

#### Cryptographic Token Generation ([[entities/src-auth]])
The core authentication module will differentiate token issuance payload types by introducing explicit `type` claims into the JWT structure to prevent token reuse across boundaries (preventing an access token from being used as a refresh token or vice-versa):

```python
# src/auth.py additions
def create_refresh_token(data: dict, expires_delta: timedelta = None) -> str:
    to_encode = data.copy()
    expire = datetime.utcnow() + (expires_delta or timedelta(days=settings.REFRESH_TOKEN_EXPIRE_DAYS))
    to_encode.update({
        "exp": expire,
        "type": "refresh"
    })
    return jwt.encode(to_encode, settings.JWT_SECRET, algorithm=settings.ALGORITHM)
```

#### API Delivery Layer Router Adjustments ([[entities/src-main]])
1. **`/api/v1/auth/login`**:
   Accepts credentials using form-urlencoded values (see [[decisions/oauth2-form-integration]] and [[concepts/authentication-flows]]). Instead of returning both tokens in the JSON response payload, it will return the short-lived access token in the JSON body and register the refresh token as an encrypted cookie.
   
2. **`/api/v1/auth/refresh`**:
   A dedicated endpoint that reads the incoming `refresh_token` cookie from the client's request. It parses the token payload, verifies its signature and cryptographic validity, checks that the `type` claim equals `"refresh"`, and returns a brand-new short-lived access token in the JSON body.

---

## Consequences

### Positive Impacts
* **Drastically Reduced Attack Surface**: If an attacker executes an XSS attack, they can only compromise the in-memory access token, which automatically expires within 15 minutes. They cannot exfiltrate the HTTPOnly refresh token.
* **Frictionless User Experience**: Frontend clients can implement dynamic, silent token-refresh logic in interceptor layers. When an access token expires, the client requests a new one in the background without prompting the user.
* **Minimized Ambient Authority Risk**: Restricting the cookie scope to `Path=/api/v1/auth/refresh` prevents the browser from automatically sending the cookie with standard API operations, preserving the performance of stateless headers.

### Negative Impacts / Trade-offs
* **Strict CORS & Domain Requirements**: Deploying cookies requires precise domain alignments. If the frontend client and the **Mini AuthService** are hosted on different top-level domains, configuring cross-origin cookies requires setting `SameSite=None` and relying completely on `Secure` attributes, which can run into varying browser policies.
* **Increased Complexity**: Requires client-side developers to configure HTTP client interceptors (such as Axios or Fetch interceptors) to catch `401 Unauthorized` responses and initiate the silent-refresh handshake.
* **Mitigated Revocation Capability**: Although refresh tokens are highly secure, they are still stateless under this design. If a refresh token must be revoked immediately (e.g., upon user logout or password reset), we will eventually need to track invalidated refresh tokens using a transient state store, such as Redis (see [[decisions/stateless-session-tradeoff]]).