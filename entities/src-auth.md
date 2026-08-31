<!-- anchor: src/auth.py:L1-L100 sha:HEAD -->

# Core Authentication Engine (`src/auth.py`)

The `src/auth.py` module is the cryptographic and logical core of the Mini AuthService. It is responsible for verifying user credentials, generating securely signed JSON Web Tokens (JWT), and parsing inbound tokens to establish stateless identity context. 

This component acts as a downstream worker for the API routing layer defined in [[entities/src-main]] and relies on settings supplied by [[entities/src-config]].

---

## ## Responsibilities

The primary responsibilities of the `src/auth.py` module include:
* **Credential Verification**: Validating inbound usernames and passwords against stored records (currently mock credentials, slated for upgrade to a persistent database).
* **Cryptographic Token Issuance**: Creating short-lived, symmetrically signed JWT access tokens using the `HS256` algorithm.
* **Token Decoding & Verification**: Decoding incoming tokens, validating signature integrity against the shared system secret, and checking expiry claims to authenticate stateless requests.
* **Session Integrity & Expiry Enforcement**: Guaranteeing that expired, tampered, or malformed tokens fail validation gracefully, throwing structured exceptions or returning `None` up the stack.

---

## ## Dependencies

The module utilizes a minimal set of standard libraries and external packages to remain highly performant:
* **`jwt` (PyJWT)**: The primary engine utilized to serialize, deserialize, and cryptographically sign JWT payloads. See [[concepts/jwt-security]] for design details.
* **`datetime` (Standard Library)**: Used to compute UTC-based token expiration claims (`exp`).
* **`settings` from `src.config`**: Resolves core variables such as `JWT_SECRET` and `ALGORITHM`. See [[entities/src-config]] and [[concepts/configuration-management]] for further details.

```
                  +-------------------------+
                  |     src/config.py       | (Provides Settings)
                  +------------+------------+
                               |
                               v
                  +-------------------------+
                  |      src/auth.py        | (Cryptographic Core)
                  +------------+------------+
                               ^
                               | (Invokes Guards & Verifiers)
                  +------------+------------+
                  |      src/main.py        | (FastAPI Delivery Layer)
                  +-------------------------+
```

---

## Code-Level Specification

### 1. User Authentication
The `authenticate_user` function verifies raw user credentials.

```python
def authenticate_user(username: str, password: str):
    if username == "admin" and password == "secret123":
        return {"username": "admin", "role": "admin"}
    return None
```

* **Execution Logic**: Compares input credentials against hardcoded fallback strings. If the match is successful, it returns a dict outlining user parameters; otherwise, it returns `None`.
* **Security Note**: This hardcoded strategy is highly insecure. See [[decisions/database-credential-storage]] for the architectural pathway to transition this hook to a PostgreSQL database utilizing Argon2id salted hashes.

---

### 2. Token Creation
The `create_access_token` function signs a dictionary payload with the application's symmetric key.

```python
def create_access_token(data: dict, expires_delta: timedelta = None):
    to_encode = data.copy()
    expire = datetime.utcnow() + (expires_delta or timedelta(minutes=15))
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, settings.JWT_SECRET, algorithm=settings.ALGORITHM)
```

* **Execution Logic**: 
  1. Clones the input payload (`data`) to avoid modifying memory references in-place.
  2. Calculates the token expiration stamp (`exp`) using UTC. If no `expires_delta` override is provided, it defaults to **15 minutes**.
  3. Formats the signature using the HS256 algorithm and the centralized `settings.JWT_SECRET` retrieved from [[entities/src-config]].
* **Architectural Decisions**: 
  * Symmetric signing (`HS256`) was chosen to reduce computational overhead. See [[decisions/symmetric-vs-asymmetric-signing]] for trade-offs against asymmetric schemes (like RS256).
  * Short lifetimes mitigate theft risks, although introducing long-lived cookies is recommended. See [[decisions/token-lifecycle-refresh]] for details.

---

### 3. Token Verification
The `verify_token` function attempts to decode, verify, and return the payload claims from an active token.

```python
def verify_token(token: str):
    try:
        payload = jwt.decode(token, settings.JWT_SECRET, algorithms=[settings.ALGORITHM])
        return payload
    except Exception:
        return None
```

* **Execution Logic**: 
  1. Accepts an incoming JWT string forwarded by the dependency injection system (see [[concepts/fastapi-dependency-injection]]).
  2. Decodes the payload using PyJWT, which automatically verifies signature authenticity and checks the expiration (`exp`) claim.
  3. If the token is valid, it returns the dictionary payload (typically containing the `sub` claim).
  4. If any exception is encountered (e.g., token has expired, signature is invalid, payload is malformed), the function catches it and safely returns `None`.
* **State Management**: This logic relies purely on cryptographic validation, aligning with the rules of [[concepts/stateless-sessions]] and avoiding database hits for token lookups.

---

## Architectural Context and Security Hardening

This engine is central to the core execution loops of the microservice. Below are security postures and enhancement pathways linked to this module:

1. **Integrating Token Revocation**:
   * Currently, validated tokens cannot be revoked before they hit their naturally scheduled expiration time (`exp`).
   * Adopting a stateless architecture creates a trade-off where immediate logout is difficult. See [[decisions/stateless-session-tradeoff]] for options regarding token-revocation storage lists.
2. **User Identity Flow**:
   * The returned dictionary from `verify_token` maps directly to the HTTP context inside [[entities/src-main]] to enable resource authorization. For step-by-step details on this pipeline, refer to [[concepts/authentication-flows]].