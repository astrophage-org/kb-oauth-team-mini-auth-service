# Authentication Flows

The **Mini AuthService** operates on standard OAuth2 protocols to deliver secure, stateless, and lightweight authentication. This page provides a deep technical analysis of the sequential logic, cryptographic verification, and state transitions that occur during the two primary system operations: **Flow A (Credential-Based Login)** and **Flow B (Protected Resource Authorization)**.

For a broader perspective on how these components integrate into the overall system layout, refer to the [[summaries/architectural-overview]].

---

## System Interaction Topology

The interaction pattern is designed to isolate security logic from routing overhead. The HTTP delivery layer [[entities/src-main]] leverages FastAPI's dependency injection container to delegate cryptographic operations to [[entities/src-auth]], which relies on [[entities/src-config]] for its parameters.

```
+---------------+              +---------------------+              +-----------------------+
|  HTTP Client  | <==========> |  API Routing Layer  | <==========> | Authentication Engine |
|               | (HTTP/REST)  |  (entities/src-main)| (In-Process) |  (entities/src-auth)  |
+---------------+              +---------------------+              +-----------------------+
```

---

## Flow A: Credential-Based Login (Authentication)

This flow details how an unauthenticated user or client application exchanges plain-text credentials for a cryptographically signed, short-lived JSON Web Token (JWT).

### Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    actor Client as HTTP Client
    participant API as FastAPI Router (src/main.py)
    participant Auth as Auth Engine (src/auth.py)
    participant Config as Settings (src/config.py)
    
    Client->>API: POST /api/v1/auth/login (application/x-www-form-urlencoded)
    Note over Client,API: Form Data: username, password
    
    API->>API: Parse OAuth2PasswordRequestForm
    API->>Auth: authenticate_user(username, password)
    
    alt Credentials Match ("admin" / "secret123")
        Auth-->>API: Return Identity Context {"username": "admin", "role": "admin"}
        API->>Auth: create_access_token(data={"sub": "admin"})
        Auth->>Config: Read settings.JWT_SECRET & settings.ALGORITHM
        Config-->>Auth: Secret key & "HS256"
        Auth->>Auth: Set exp = utcnow() + 15 minutes
        Auth->>Auth: Cryptographically Sign Token (HS256)
        Auth-->>API: Signed JWT (Base64URL-encoded String)
        API-->>Client: HTTP 200 OK: {"access_token": "...", "token_type": "bearer"}
    else Credentials Invalid
        Auth-->>API: Return None
        API->>API: Raise HTTPException (401 Unauthorized)
        API-->>Client: HTTP 401 Unauthorized: {"detail": "Incorrect username or password"}
    end
```

### Detailed Step-by-Step Logic

#### 1. Request Dispatch
The client issues an HTTP `POST` request to the `/api/v1/auth/login` endpoint. Following the standardized specifications in [[decisions/oauth2-form-integration]], the request payload must be transmitted as form-encoded data (`application/x-www-form-urlencoded`) rather than JSON. The payload fields include:
*   `username`: The user identifier (e.g., `"admin"`).
*   `password`: The raw string credential (e.g., `"secret123"`).

#### 2. Model Ingestion & Parameter Validation
The API layer, configured in [[entities/src-main]], intercepts the request via FastAPI's `OAuth2PasswordRequestForm` class dependency. The framework ensures:
*   The payload format matches `application/x-www-form-urlencoded`.
*   Required parameters are present, raising an automatic `HTTP 422 Unprocessable Entity` on omission.

#### 3. Identity Verification
The router invokes `authenticate_user(username, password)` from the core engine defined in [[entities/src-auth]]:
```python
def authenticate_user(username: str, password: str):
    if username == "admin" and password == "secret123":
        return {"username": "admin", "role": "admin"}
    return None
```
*   **Security Evaluation**: In its baseline configuration, credentials are verified against a hardcoded context. In standard production deployments, this engine is modified to verify salted hashes in a persistent engine, as discussed in [[decisions/database-credential-storage]].
*   **Response Output**:
    *   On a positive match, the engine yields an identity dictionary.
    *   On mismatch, it yields `None`.

#### 4. Cryptographic Token Construction
Upon receipt of a valid identity payload, the router delegates token creation to `create_access_token(data={"sub": user["username"]})`:
```python
def create_access_token(data: dict, expires_delta: timedelta = None):
    to_encode = data.copy()
    expire = datetime.utcnow() + (expires_delta or timedelta(minutes=15))
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, settings.JWT_SECRET, algorithm=settings.ALGORITHM)
```
*   **Claim Generation**: The standard claims structure includes:
    *   `sub` (Subject): Denotes the identity of the user.
    *   `exp` (Expiration Time): Enforced at 15 minutes from creation.
*   **Signing Mechanics**: The dictionary payload is serialized to JSON and cryptographically signed using the symmetric **HS256** algorithm and the `JWT_SECRET` retrieved from [[entities/src-config]]. The trade-offs of using symmetric cryptosystems over asymmetric options are fully analyzed in [[decisions/symmetric-vs-asymmetric-signing]].

#### 5. HTTP Response Serializing
The generated token string is encapsulated in a standardized OAuth2 payload structure:
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "bearer"
}
```
This is dispatched to the client with an HTTP Status Code `200 OK`. If authentication fails, the server halts execution by raising an `HTTPException(status_code=401, detail="Incorrect username or password")`.

---

## Flow B: Protected Resource Request (Authorization)

This flow illustrates how incoming requests are authenticated in a stateless environment. Rather than validating session files on disk or checking a centralized database on every transaction, authorization is derived dynamically from cryptographic tokens.

### Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    actor Client as HTTP Client
    participant API as FastAPI Router (src/main.py)
    participant Guard as verify_token Dependency
    participant Auth as Auth Engine (src/auth.py)
    participant Config as Settings (src/config.py)
    
    Client->>API: GET /api/v1/auth/me (Headers: Authorization: Bearer <token>)
    Note over API: FastAPI interceptor invokes verify_token Dependency
    API->>Guard: Execute Depends(verify_token)
    
    Guard->>Auth: verify_token(token_string)
    Auth->>Config: Read settings.JWT_SECRET & settings.ALGORITHM
    Config-->>Auth: Secret key & "HS256"
    Auth->>Auth: Cryptographically Decode and Verify Signature & Expiry
    
    alt Token Cryptographically Valid
        Auth-->>Guard: Return Decoded Payload {"sub": "admin", "exp": 1716298200}
        Guard-->>API: Inject Decoded Token Payload
        API->>API: Process Request with Verified Context
        API-->>Client: HTTP 200 OK: {"username": "admin", "status": "active"}
    else Token Expired, Tampered, or Malformed
        Auth-->>Guard: Return None
        Guard->>API: Raise HTTPException (401 Unauthorized)
        API-->>Client: HTTP 401 Unauthorized: Invalid or expired token
    end
```

### Detailed Step-by-Step Logic

#### 1. Request Submission with Bearer Token
The client initiates a request to the protected endpoint `/api/v1/auth/me` with the `Authorization` header populated with the JWT:
```http
GET /api/v1/auth/me HTTP/1.1
Host: localhost:8000
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

#### 2. Declarative Dependency Injection Interception
The router defines the endpoint handler with a strict, injected dependency constraint:
```python
@app.get("/api/v1/auth/me")
async def get_current_user(token_data: dict = Depends(verify_token)):
```
Before the controller logic executes, FastAPI resolves `Depends(verify_token)`, pausing controller execution. This modular execution structure is detailed in [[concepts/fastapi-dependency-injection]].

#### 3. Cryptographic Decode and Signature Verification
The dependency invokes the validation core inside [[entities/src-auth]]:
```python
def verify_token(token: str):
    try:
        payload = jwt.decode(token, settings.JWT_SECRET, algorithms=[settings.ALGORITHM])
        return payload
    except Exception:
        return None
```
The library attempts to decode and verify the token. This process involves:
*   **Signature Verification**: Computing the HS256 signature of the token's header and payload using the local `JWT_SECRET` and comparing it to the signature attached to the incoming token. If any character has changed, verification fails.
*   **Expiration Verification**: Comparing the value of the `exp` claim against the current system time (in UTC). If the current time is greater than `exp`, a `ExpiredSignatureError` is caught.
*   **Claim Parsing**: Extracting the `sub` claim to define the authenticated context.

For an extensive analysis of the payload validation mechanics and underlying security, see [[concepts/jwt-security]].

#### 4. Authorization Enforcement & Stateless Context Mapping
*   **On Failure**: If the cryptographic signature is invalid, expired, or malformed, `verify_token` returns `None` or raises an error. The dependency wrapper translates this state into an `HTTP 401 Unauthorized` exception, terminating execution before the route handler is invoked.
*   **On Success**: The parsed payload dictionary is injected into the controller argument `token_data`. The controller accesses this context state directly:
    ```python
    return {"username": token_data["sub"], "status": "active"}
    ```
    This completes the lifecycle of a **Stateless Session**, as further documented in [[concepts/stateless-sessions]]. Identity is derived cryptographically from the token payload, with no database queries required during token verification.

---

## Architectural Security Context & Recommended Hardening

While these sequential flows establish a robust OAuth2 baseline, production deployments must address key trade-offs and implement additional hardening steps:

1. **Stateless Token Revocation Limits**: Since authorization does not query a central database, tokens cannot be revoked on logout or credential reset without introducing a storage layer (e.g., a Redis-backed denylist). The balance between stateless scaling and revocation control is detailed in [[decisions/stateless-session-tradeoff]].
2. **Secret Management Hardening**: Storing the `JWT_SECRET` in hardcoded static classes (such as the default `Settings` in [[entities/src-config]]) presents a security risk. Transitioning to a production posture requires adopting the 12-Factor methodology by loading secrets from environment variables, as described in [[concepts/configuration-management]].
3. **Database-backed Password Verification**: Production applications must upgrade from static hardcoded strings to secure database verification. This process should connect to a PostgreSQL database and use modern, standard hashing techniques like **Argon2id** (see [[decisions/database-credential-storage]]).
4. **Token Lifecycle Expansion**: Utilizing a 15-minute access token limit requires a secure mechanism for session continuation. Deploying an `HTTPOnly` cookie-based refresh token framework provides continuous, secure sessions without requiring the user to repeatedly enter credentials, as detailed in [[decisions/token-lifecycle-refresh]].