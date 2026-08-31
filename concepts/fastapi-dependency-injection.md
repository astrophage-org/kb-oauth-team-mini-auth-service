# FastAPI Dependency Injection: Modular Authentication Guards

In the **Mini AuthService**, the API delivery layer is designed to be lean and decoupled from cryptographic validation and identity resolution. This segregation of concerns is achieved by utilizing FastAPI’s declarative **Dependency Injection (DI)** container. 

By utilizing FastAPI's `Depends` engine, authorization guards are implemented as modular, reusable callables. This architecture simplifies route handler code, isolates cryptographic validation logic, and enables declarative access-control policies across endpoints.

---

## The Role of Dependency Injection in Authentication

FastAPI's dependency injection framework resolves requirements before executing route handlers. In a stateless microservice like the Mini AuthService, DI is used to:

1. **Intercept Incoming Requests**: Extract and parse security tokens from HTTP headers.
2. **Enforce Security Policies**: Automatically validate cryptographic signatures, verify token expiration, and check scopes/roles.
3. **Inject Context**: Provide the resolved identity context (e.g., username, claims) directly to the route handler as a function parameter.
4. **Short-Circuit Execution**: Reject invalid requests immediately with a standard HTTP error code, preventing downstream business logic from executing.

This flow is illustrated in [[summaries/architectural-overview]] under **Flow B: Protected Resource Request**.

---

## Architectural Analysis: Current Implementation

The current codebase leverages dependency injection in two critical areas: the login form parser and the session verification guard.

### 1. Form-Based Login Dependency
The `/api/v1/auth/login` endpoint uses FastAPI's `Depends` to parse incoming OAuth2 compliance forms:

```python
# From src/main.py
@app.post("/api/v1/auth/login")
async def login(form_data: OAuth2PasswordRequestForm = Depends()):
    user = authenticate_user(form_data.username, form_data.password)
    ...
```

By declaring `form_data: OAuth2PasswordRequestForm = Depends()`, the router delegates form parsing, validation, and schema compliance to FastAPI's built-in OAuth2 structures. This standardizes compliance with the OAuth2 spec, as discussed in [[decisions/oauth2-form-integration]].

### 2. The Current Route Guard Dependency
The `/api/v1/auth/me` endpoint enforces token authentication using `verify_token`:

```python
# From src/main.py
@app.get("/api/v1/auth/me")
async def get_current_user(token_data: dict = Depends(verify_token)):
    return {"username": token_data["sub"], "status": "active"}
```

The dependency `verify_token` is imported from [[entities/src-auth]]:

```python
# From src/auth.py
def verify_token(token: str):
    try:
        payload = jwt.decode(token, settings.JWT_SECRET, algorithms=[settings.ALGORITHM])
        return payload
    except Exception:
        return None
```

---

## Critical Evaluation & Vulnerabilities in the Current DI Design

While the architectural *intent* of using modular dependencies is sound, a deep-dive analysis of the current implementation in `src/main.py` and `src/auth.py` reveals two critical architectural gaps that must be addressed before production deployment.

### Gap A: The "NoneType" Unhandled Exception Vulnerability
If an expired or malformed token is provided, `verify_token` catches the exception and returns `None`:

```python
except Exception:
    return None
```

In `src/main.py`, the dependency `get_current_user` receives this `None` value as `token_data` and immediately attempts to access `token_data["sub"]`:

```python
@app.get("/api/v1/auth/me")
async def get_current_user(token_data: dict = Depends(verify_token)):
    return {"username": token_data["sub"], "status": "active"}
```

* **The Failure**: When `token_data` is `None`, Python raises a `TypeError: 'NoneType' object is not subscriptable`. 
* **The Result**: FastAPI returns an unhandled **HTTP 500 Internal Server Error** instead of a clean, secure **HTTP 401 Unauthorized** response. This leaks execution details and violates security best practices.

### Gap B: Missing Header Parsing Scheme
In the current code, `verify_token(token: str)` expects `token` to be passed directly as a function argument. 

Because it does not utilize FastAPI's `OAuth2PasswordBearer` or `HTTPBearer` security schemes, FastAPI's routing layer cannot automatically resolve the token from the `Authorization: Bearer <JWT>` header. Instead, FastAPI will default to looking for a query parameter or request body field named `token` (e.g., `/api/v1/auth/me?token=...`). This violates stateless design principles and standard browser headers (see [[concepts/stateless-sessions]] and [[decisions/stateless-session-tradeoff]]).

---

## Hardening the Guard: The Refactoring Blueprint

To address these vulnerabilities, we must restructure the dependency injection mechanism to handle header extraction automatically and short-circuit failure states correctly by raising an `HTTPException` directly from within the dependency function.

### The Standardized Refactored Pattern

The implementation below demonstrates how to integrate `OAuth2PasswordBearer` to locate the Bearer token automatically and short-circuit unauthorized requests at the DI layer:

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
import jwt
from .config import settings

# 1. Instantiate the standard OAuth2 scheme, defining the token retrieval URL.
# This instructs FastAPI to read the 'Authorization: Bearer <token>' header.
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/v1/auth/login")

def get_verified_user_payload(token: str = Depends(oauth2_scheme)) -> dict:
    """
    Dependency Guard: Resolves the JWT, verifies its signature and expiration,
    and returns the claims. Short-circuits with HTTP 401 on validation failure.
    """
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        # Cryptographically decode the payload (see [[concepts/jwt-security]])
        payload = jwt.decode(
            token, 
            settings.JWT_SECRET, 
            algorithms=[settings.ALGORITHM]
        )
        username: str = payload.get("sub")
        if username is None:
            raise credentials_exception
        return payload
    except jwt.PyJWTError:
        # Catch any signing, expiration, or format errors and return a clean 401
        raise credentials_exception
```

### Implementing the Hardened Guard in Router Endpoints
With the refactored guard, the API route handlers in [[entities/src-main]] remain completely declarative and free of error-handling overhead:

```python
@app.get("/api/v1/auth/me")
async def get_current_user(token_data: dict = Depends(get_verified_user_payload)):
    # Reaching this point guarantees token_data is valid, non-None, and verified
    return {"username": token_data["sub"], "status": "active"}
```

This guarantees that:
* If the token is invalid, the client receives a proper `401 Unauthorized` before the endpoint is called.
* No internal `TypeError` crashes can occur inside the route logic.
* Decrypting and validating identity context is completely isolated from HTTP routing concerns.

---

## Benefits of the Dependency Injection Guard Pattern

| Benefit | Description | Related Architecture Link |
| :--- | :--- | :--- |
| **Declarative Security** | Developers secure any endpoint simply by adding `Depends(get_verified_user_payload)` as a route parameter. | [[entities/src-main]] |
| **Separation of Concerns** | Cryptographic token decoding is isolated in the dependency; routers only handle HTTP requests and serialization. | [[entities/src-auth]] |
| **Stateless Identity** | Reconstructs user context purely from the cryptographic payload, preserving stateless scalability. | [[concepts/stateless-sessions]] |
| **Simplified Unit Testing** | Dependencies can be globally overridden using FastAPI’s `app.dependency_overrides` map, allowing easy mocking of auth contexts during testing. | [[index]] |