# Configuration Management & 12-Factor Compliance

This document outlines the architectural patterns and implementation details for upgrading the Mini AuthService configuration engine from hardcoded settings to a 12-Factor compliant, type-safe schema powered by **Pydantic Settings**.

---

## 1. Introduction to 12-Factor Configuration

The [12-Factor App methodology](https://12factor.net/) asserts a strict separation of config from code. Config includes everything that is likely to vary between deploys (staging, production, developer environments, etc.). This includes:
* Database connection URIs (`DATABASE_URL`)
* Secret keys and cryptographic materials (`JWT_SECRET`)
* System parameters and performance tunings (`TOKEN_EXPIRY_MINUTES`)

### The Anti-Pattern: Hardcoded Secrets
In the legacy implementation of `[[entities/src-config]]`, parameters were stored directly as class attributes in a standard Python class:

```python
# Legacy src/config.py
class Settings:
    JWT_SECRET: str = "astrophage-secret-key-123"
    ALGORITHM: str = "HS256"
    DATABASE_URL: str = "postgresql://localhost:5432/auth_db"

settings = Settings()
```

This approach presents significant security and operational risks:
1. **Secret Leakage**: Committing credentials to version control (Git) exposes production environments.
2. **Immutability of Environments**: Changing a database connection or rotating a secret requires a full source-code modification and redeployment, violating basic CI/CD principles.
3. **Lack of Type Safety**: Environment-specific overrides cannot be validated automatically at application startup.

---

## 2. Upgrading to Pydantic Settings

To transition the service to production-grade compliance, we leverage `pydantic-settings` (v2). This provides automatic validation, environment variable binding, and explicit type conversion during the application's startup sequence.

### Key Benefits of Pydantic Settings
* **Fail-Fast Bootstrapping**: The application refuses to initialize if critical environment variables (such as a database connection string) are missing or misconfigured. This prevents half-broken containers from entering a running state.
* **Type Coercion**: Converts incoming environment string variables to native Python objects (e.g., checking if integer ports, booleans, or connection URIs match structured types).
* **Multiple Source Resolution**: Seamlessly merges default parameters with values from `.env` files, environment values exported in bash, or environment overrides in Kubernetes manifests.

---

## 3. Upgraded Configuration Engine Implementation

The refactored configuration engine is declared in `[[entities/src-config]]`. Below is the complete, production-ready schema:

```python
# src/config.py
from typing import Literal
from pydantic import Field, PostgresDsn
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    """
    Centralized Configuration Manager for Mini AuthService.
    Conforms to 12-Factor App principles by parsing and validating 
    configuration directly from environment variables.
    """
    
    # --- Cryptographic & Security Settings ---
    # No default value provided for production; it must be supplied in the environment.
    JWT_SECRET: str = Field(
        ...,
        description="High-entropy secret key for HS256 symmetric signing.",
        validation_alias="JWT_SECRET"
    )
    
    # Restrict algorithm choice to secure symmetric schemes evaluated in [[decisions/symmetric-vs-asymmetric-signing]].
    ALGORITHM: Literal["HS256", "HS384", "HS512"] = Field(
        default="HS256",
        description="Symmetric algorithm used to sign JWT tokens."
    )
    
    ACCESS_TOKEN_EXPIRE_MINUTES: int = Field(
        default=15,
        description="Lifespan of the issued access token.",
        gt=0
    )

    # --- Persistence Settings ---
    # Leveraging Pydantic's native PostgresDsn for syntax verification.
    DATABASE_URL: PostgresDsn = Field(
        ...,
        description="Connection URI for the primary PostgreSQL database.",
        validation_alias="DATABASE_URL"
    )

    # --- Pydantic Settings Model Configuration ---
    model_config = SettingsConfigDict(
        # Look for a local '.env' file first for development ease
        env_file=".env",
        env_file_encoding="utf-8",
        # Ignore extra system environment variables not declared in this class
        extra="ignore",
        # Case-insensitive environment variable lookup
        case_sensitive=False
    )

# Instantiate as a single source of truth (SSOT) singleton
settings = Settings()
```

---

## 4. Configuration Binding & Resolution Order

Pydantic Settings scans configurations using a precise hierarchy. Overrides are evaluated in the following order (from highest priority to lowest):

1. **Explicit Constructor Arguments**: `Settings(JWT_SECRET="override")` (useful for unit tests).
2. **System Environment Variables**: Values exported via shell or Docker runtime environments (e.g., `export JWT_SECRET="prod-secret"`).
3. **Environment Files**: Entries specified within the targeted `.env` file.
4. **Default Fields**: The fallback values defined directly in the Pydantic model declaration (e.g., `ALGORITHM="HS256"`).

```
                      +-----------------------------+
                      | Explicit Constructor Args   | (Highest Priority)
                      +--------------+--------------+
                                     |
                                     v
                      +--------------+--------------+
                      | System Environment Vars     |
                      +--------------+--------------+
                                     |
                                     v
                      +--------------+--------------+
                      |    .env Configuration File  |
                      +--------------+--------------+
                                     |
                                     v
                      +--------------+--------------+
                      |     Fallback Class Defaults | (Lowest Priority)
                      +-----------------------------+
```

### Local Development Environment Setup (`.env`)
For local execution, create a `.env` file at the project root. This file should be explicitly added to `.gitignore` to prevent leakage into shared code repositories:

```ini
# .env (Do not commit to Version Control)
JWT_SECRET=super-secure-local-development-secret-key-32-chars-long
ALGORITHM=HS256
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/auth_db
ACCESS_TOKEN_EXPIRE_MINUTES=15
```

---

## 5. Downstream System Integration

All dependencies and downstream domains call this configuration singleton. The runtime configuration flows directly into the core execution engines.

### Authentication Core Integration
`[[entities/src-auth]]` references the config singleton to sign and verify payloads dynamically.

```python
# src/auth.py (Excerpt)
import jwt
from datetime import datetime, timedelta
from .config import settings  # Import the single source of truth settings

def create_access_token(data: dict, expires_delta: timedelta = None):
    to_encode = data.copy()
    expire = datetime.utcnow() + (expires_delta or timedelta(minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES))
    to_encode.update({"exp": expire})
    # Dynamically references validated settings
    return jwt.encode(to_encode, settings.JWT_SECRET, algorithm=settings.ALGORITHM)

def verify_token(token: str):
    try:
        # Dynamically uses the validated algorithm and secret
        payload = jwt.decode(token, settings.JWT_SECRET, algorithms=[settings.ALGORITHM])
        return payload
    except Exception:
        return None
```

### Persistence Integration
The `DATABASE_URL` is parsed directly into database connection factories as described in `[[decisions/database-credential-storage]]`. If the connection string is malformed (e.g., contains standard typo protocols like `postgesql://`), Pydantic's `PostgresDsn` type validation will crash immediately at container boot, preventing runtime failures.

---

## 6. Architectural Recommendations & Production Hardening

To maintain a secure operational posture, apply these practices when running the upgraded configuration system:

1. **Zero Defaults for Secrets**: Ensure that properties like `JWT_SECRET` and `DATABASE_URL` use the Pydantic ellipses (`...`) to denote *required fields*. If these are missing from the environment, the service must fail to launch.
2. **Avoid Global Imports inside Handlers**: Rely on importing `settings` from `src.config` as a module attribute or leverage FastAPI's dependency injection container (`[[concepts/fastapi-dependency-injection]]`) to inject settings into endpoints. Injecting settings simplifies mocking environments during end-to-end integration testing.
3. **Secure secret injection**: In Kubernetes environments, do not expose secrets as plain-text environment variables in the pod specification. Instead, inject them as environment variables sourced from Kubernetes `Secrets` resources or dynamic engines like HashiCorp Vault.