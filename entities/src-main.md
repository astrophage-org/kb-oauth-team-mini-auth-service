<!-- anchor: src/main.py:L1-L100 sha:HEAD -->

# API Delivery Layer Specification (`src/main.py`)

The `src/main.py` file serves as the entrypoint for the **Mini AuthService** application. It implements the FastAPI delivery layer, setting up routing, request parsing, input validation, HTTP exception mapping, and response serialization. 

This file acts as the boundary layer of the microservice, exposing REST endpoints conforming to the OAuth2 standard and dispatching core cryptographic operations to the authentication engine.

---

## Responsibilities

The primary responsibilities of the FastAPI delivery layer include:
* **Application Lifecycle Management**: Initializing the main `FastAPI` application instance with metadata (title, version).
* **Routing & Endpoint Presentation**: Exposing the public REST API surface, specifically the `/api/v1/auth/login` and `/api/v1/auth/me` routes.
* **Input Deserialization & Protocol Conformity**: Using `OAuth2PasswordRequestForm` to enforce compliant parsing of `application/x-www-form-urlencoded` payloads as detailed in [[decisions/oauth2-form-integration]].
* **Authentication Guard Application**: Utilizing FastAPI's dependency injection container (`Depends`) to inject verification filters and handle access control before routing hits underlying business logic. See [[concepts/fastapi-dependency-injection]].
* **HTTP Status Code Mapping**: Translating security results (e.g., identity verification failure) into proper RFC-compliant HTTP status codes (such as `401 Unauthorized`).

---

## Dependencies

The implementation leverages both framework utilities and internal modules:

* **FastAPI Framework**:
  * `FastAPI`: The main application orchestrator.
  * `Depends`: The declarative dependency injection tool used to lock down endpoints. See [[concepts/fastapi-dependency-injection]].
  * `HTTPException` & `status`: Clean exception propagation structures for RFC compliance.
  * `OAuth2PasswordRequestForm`: Ensures OAuth2 compliant form handling. See [[decisions/oauth2-form-integration]].
* **Internal Security Engine** (`src/auth.py`):
  * `authenticate_user`: Verifies client-provided credentials against stored accounts. See [[entities/src-auth]].
  * `create_access_token`: Generates short-lived, cryptographically signed HS256 JWT tokens. See [[concepts/jwt-security]].
  * `verify_token`: Cryptographically validates the incoming bearer JWT to extract state. See [[concepts/stateless-sessions]].
* **Configuration Manager** (`src/config.py`):
  * `settings`: Retrieves environment configuration variables (e.g., token expiration, secrets). See [[entities/src-config]] and [[concepts/configuration-management]].

---

## Code-level Endpoint Specification

### 1. User Login and JWT Issuance
* **Endpoint**: `POST /api/v1/auth/login`
* **Format**: `application/x-www-form-urlencoded`
* **Payload Structure**: Enforced by `OAuth2PasswordRequestForm` (requires `username` and `password` fields).
* **Behavior**:
  1. The endpoint intercepts the form submission and passes the values directly to the authentication layer: `authenticate_user(form_data.username, form_data.password)`.
  2. If the user authentication returns `None`, a `401 Unauthorized` exception is raised with the detail message `"Incorrect username or password"`.
  3. If authentication is successful, the route initiates JWT generation using `create_access_token()`, passing the identity context (`{"sub": user["username"]}`) to be cryptographically embedded in the token's payload.
  4. Returns a standard OAuth2 payload containing the raw JWT and the token type:
     ```json
     {
       "access_token": "<JWT_STRING>",
       "token_type": "bearer"
     }
     ```

For a sequential breakdown of this process, see the diagram under **Flow A** in [[summaries/architectural-overview]] and review [[concepts/authentication-flows]].

### 2. Protected Resource Access (Identity Recovery)
* **Endpoint**: `GET /api/v1/auth/me`
* **Format**: Standard HTTP Authorization Header (`Authorization: Bearer <token>`)
* **Behavior**:
  1. This path relies on FastAPI's dependency injection container to evaluate access permission *before* executing the route body:
     ```python
     async def get_current_user(token_data: dict = Depends(verify_token)):
     ```
  2. The callable dependency `verify_token` receives the HTTP Authorization header, verifies the signature using the HS256 symmetric algorithm, handles validation constraints, and yields the decoded claims payload to the controller. See [[decisions/symmetric-vs-asymmetric-signing]] for details on this cryptographic trade-off.
  3. If token verification fails (expired, altered signature, missing claims), the request is automatically rejected.
  4. If validation succeeds, identity metadata is dynamically derived from the `sub` claim in a stateless manner (see [[concepts/stateless-sessions]]).
  5. The path function returns the extracted identity context:
     ```json
     {
       "username": "admin",
       "status": "active"
     }
     ```

For a sequential breakdown of this validation flow, see **Flow B** in [[summaries/architectural-overview]].

---

## Future Hardening Recommendations

The delivery layer code implemented in `src/main.py` requires several enhancements to achieve modern production readiness:

1. **Strict Content-Type Validation and JSON Login Options**: While using `OAuth2PasswordRequestForm` standardizes input (see [[decisions/oauth2-form-integration]]), modern clients frequently request standard application/json requests. The delivery layer should support both form-encoded and JSON-body payloads.
2. **Refresh Token Pipeline Support**: The layer currently relies solely on short-lived tokens. It must be updated to deploy secure `HTTPOnly` cookies containing cryptographically paired refresh keys. See [[decisions/token-lifecycle-refresh]].
3. **Database Guard Transition**: The endpoints must be modified to read from verified database queries rather than running mock credentials logic. See [[decisions/database-credential-storage]].
4. **Rate Limiting**: Integrate rate-limiting middleware on the `/api/v1/auth/login` route to prevent credential stuffing and brute-force attacks.