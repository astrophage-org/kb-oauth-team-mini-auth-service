<!-- anchor: src/auth.py:L1-L100 sha:HEAD -->

# Configuration Manager: `src/config.py`

The `src/config.py` file serves as the Single Source of Truth (SSOT) for the global state, secrets, and integration endpoints of the Mini AuthService. It initializes the application-wide environment parameters required for cryptographic signing, token lifecycles, and backend integrations.

---

## Responsibilities

The primary responsibilities of the configuration layer are:

*   **Centralized State Management**: Acting as the unified configuration repository accessed by both the API layer (`src/main.py`) and the security core (`src/auth.py`).
*   **Cryptographic parameter store**: Providing the secret keys and algorithms needed to establish the security posture for symmetric JWT signing.
*   **Database Endpoint Exposure**: Provisioning connection strings required to migrate the service from in-memory mock data to persistent relational databases.

---

## Dependencies

The current minimal version of the configuration module depends only on standard Python types, but it is heavily referenced across the codebase:

*   **Consumed by [[entities/src-auth]]**: Imports `settings.JWT_SECRET` and `settings.ALGORITHM` to sign and verify JSON Web Tokens (JWT).
*   **Consumed by [[entities/src-main]]**: Imports the global `settings` instance to coordinate metadata configuration, routes, and logging.
*   **Target of refactoring via [[concepts/configuration-management]]**: Planned integration with `pydantic-settings` to support 12-factor app environment configuration.

---

## Code-Level Specification

The current implementation initializes a basic Python class mapping static values to instance variables, instantiating a global singleton:

```python
class Settings:
    JWT_SECRET: str = "astrophage-secret-key-123"
    ALGORITHM: str = "HS256"
    DATABASE_URL: str = "postgresql://localhost:5432/auth_db"

settings = Settings()
```

### Config Variable Analysis

*   **`JWT_SECRET`**:
    *   *Type*: `str`
    *   *Current Value*: `"astrophage-secret-key-123"`
    *   *Role*: Used as the pre-shared key (PSK) to sign tokens with the HMAC SHA-256 algorithm.
    *   *Critical Security Warning*: Under current conditions, this secret is hardcoded. This exposes the system to identity spoofing, as detailed in [[concepts/jwt-security]] and [[decisions/symmetric-vs-asymmetric-signing]].
*   **`ALGORITHM`**:
    *   *Type*: `str`
    *   *Default*: `"HS256"`
    *   *Role*: Sets the symmetric encryption algorithm used by PyJWT.
*   **`DATABASE_URL`**:
    *   *Type*: `str`
    *   *Default*: `"postgresql://localhost:5432/auth_db"`
    *   *Role*: Prepared endpoint to transition user data storage away from plain text code files towards a database, in line with [[decisions/database-credential-storage]].

---

## Security Deficiencies and Production Refactoring

The current implementation of `src/config.py` does not comply with modern security principles. Hardcoding credentials in version control poses a major compromise risk.

As discussed in [[concepts/configuration-management]], the system should be refactored to use Pydantic Settings for 12-factor compliance. This change allows pulling configuration values dynamically from system environment variables or `.env` files.

### Recommended Implementation (Pydantic Settings Migration)

Below is the approved refactoring pattern to secure `src/config.py`:

```python
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic import Field

class Settings(BaseSettings):
    # Retrieve from environment, or raise validation error if missing in production
    JWT_SECRET: str = Field(
        ..., 
        description="Crucial key used to sign access and refresh tokens."
    )
    ALGORITHM: str = Field(
        default="HS256", 
        description="Cryptographic signing algorithm."
    )
    DATABASE_URL: str = Field(
        default="postgresql://localhost:5432/auth_db",
        description="Database connection URI."
    )

    # Enable reading values from local .env files automatically
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        extra="ignore"
    )

settings = Settings()
```

By transitioning to this declarative pattern:
1. Production deployments can supply `JWT_SECRET` through container orchestrator configurations (e.g., Kubernetes Secrets) without changing code.
2. Unset variables trigger explicit validation errors on startup, preventing the application from running with dangerous fallback defaults.