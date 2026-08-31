# Standardizing Inputs via OAuth2 Password Request Form Integration

## Status

**Accepted**

## Context

In standard web applications, clients typically communicate with JSON-based REST APIs by sending payloads formatted as `application/json`. However, the OAuth2 specification ([RFC 6749 Section 4.3.2](https://datatracker.ietf.org/doc/html/rfc6749#section-4.3.2)) explicitly mandates that Resource Owner Password Credentials Grant requests use the `application/x-www-form-urlencoded` format. This request must contain specific form parameters, namely `username` and `password`.

When building the **Mini AuthService** (detailed in [[summaries/architectural-overview]]), we faced a choice:
1. Accept custom JSON bodies (e.g., `{"username": "...", "password": "..."}`), which violates standard OAuth2 spec expectations.
2. Build a custom parsing dependency to support both form-urlencoded data and JSON payloads.
3. Align directly with standard OAuth2 compliance using FastAPI's built-in dependency tools.

Furthermore, integrating with interactive documentation platforms (such as Swagger UI / OpenAPI) and third-party identity tools requires adhering to standardized login inputs. FastAPI provides built-in tools to resolve this, but utilizing them changes how incoming parameters are parsed and validated.

## Decision

We decided to standardize all token-generation requests on the standard OAuth2 specification by requiring `application/x-www-form-urlencoded` payloads at the `/api/v1/auth/login` endpoint. This is achieved by declaring FastAPI's `OAuth2PasswordRequestForm` class as a dependency in our primary router (`[[entities/src-main]]`).

```python
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordRequestForm

@app.post("/api/v1/auth/login")
async def login(form_data: OAuth2PasswordRequestForm = Depends()):
    # Form data properties (form_data.username, form_data.password) are extracted
    # from the x-www-form-urlencoded body.
```

### Key Technical Mechanisms

1. **Dependency Injection Integration**: We utilize [[concepts/fastapi-dependency-injection]] to parse and validate incoming form data streams. FastAPI automatically generates the correct OpenAPI metadata to signal that this endpoint consumes form data rather than a JSON body.
2. **Schema & Argument Mapping**: The `OAuth2PasswordRequestForm` parses standard parameters:
   * `username` (mapped to our backend username lookup)
   * `password` (passed directly to the verification engine in [[entities/src-auth]])
   * Optional fields such as `scope`, `client_id`, and `client_secret` (reserved for future conformance).
3. **Authentication Flows**: The login controller coordinates standard extraction and authentication routing, as detailed in [[concepts/authentication-flows]].
4. **Credential Verification Hand-off**: Once the form data is verified, credentials are authenticated. Currently, this uses a static dictionary matching, but is architected to migrate seamlessly to an Argon2id-hashed database structure via [[decisions/database-credential-storage]].

---

### Consequences & Trade-offs

* **Pros**:
  * **Interactive Spec Support**: Swagger UI (`/docs`) natively detects the OAuth2 setup and displays an "Authorize" padlock. Developers can log in directly through the browser interface.
  * **Standard Compliance**: Instantly compatible with industry-standard OAuth2 tools, clients, and proxy authenticators without custom adapter scripts.
  * **Strict Typing**: Utilizing FastAPI's native validation guarantees that requests missing standard fields are rejected before hitting downstream security controllers in [[entities/src-auth]].

* **Cons**:
  * **Inconsistent Request Payloads**: Client developers must remember to submit authentication requests as URL-encoded form data, while the rest of the microservice architecture utilizes JSON request bodies.
  * **Response Format Invariance**: While input payloads are form-encoded, response payloads must still return a standardized JSON structure (`{"access_token": "...", "token_type": "bearer"}`) to remain compliant with standard token consumption flows.

## See Also

* [[summaries/architectural-overview]]
* [[entities/src-main]]
* [[concepts/authentication-flows]]
* [[concepts/fastapi-dependency-injection]]
* [[decisions/database-credential-storage]]
* [[decisions/token-lifecycle-refresh]]