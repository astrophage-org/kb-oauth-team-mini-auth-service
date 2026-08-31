# Architectural Decision Record: Database Credential Storage & Password Hashing

## Status

Approved

## Context

The initial proof-of-concept for the **Mini AuthService** (detailed in the [[summaries/architectural-overview]]) utilized plain-text, hardcoded user records directly inside the core authentication engine (`[[entities/src-auth]]`):

```python
def authenticate_user(username: str, password: str):
    if username == "admin" and password == "secret123":
        return {"username": "admin", "role": "admin"}
    return None
```

This implementation presents critical security vulnerabilities and operational limitations:
1. **Plain-text Credentials**: Secrets are committed to source control, violating standard security compliance.
2. **Lack of Persistence**: Real-world operations require registering, modifying, and deactivating user accounts dynamically.
3. **Inability to Scale**: A hardcoded single-user dictionary cannot support production-scale user management or tenant segregation.

To move toward production readiness, we must integrate a robust, stateful persistence layer to store user identities and verify passwords securely. As noted in `[[entities/src-config]]`, a placeholder connection string (`DATABASE_URL = "postgresql://localhost:5432/auth_db"`) was established. 

### Cryptographic Hashing Standards
When persisting user passwords, storing raw strings or utilizing obsolete cryptographic algorithms (such as MD5, SHA-1, or unsalted SHA-256) is unacceptable. Modern standards require salted, computationally expensive, memory-hard hashing functions to neutralize GPU/ASIC-accelerated offline brute-force attacks. 

We evaluated the following primary candidates:
* **PBKDF2**: Highly compatible and standardized, but vulnerable to GPU-based parallel acceleration.
* **bcrypt**: Extremely reliable and widely deployed, but lacks tunable memory parameters to combat specialized hardware (ASICs).
* **Argon2id**: The winner of the Password Hashing Competition (PHC) and recommended by OWASP. It provides tunable time cost (iterations), memory cost, and parallelization parameters, offering superior defense against both GPU and ASIC-based password cracking.

---

## Decision

We will upgrade the credential verification system from static code-defined variables to a secured **PostgreSQL** instance, storing password records protected by the **Argon2id** hashing algorithm. 

This decision is decomposed into the following implementation specifications:

### 1. Choice of Algorithm: Argon2id
We will use the Python library `argon2-cffi` to perform password hashing and validation. The default parameters will align with the OWASP cryptographic recommendations:
* **Memory profile ($m$)**: 65,536 KB (64 MiB)
* **Time profile ($t$)**: 3 iterations
* **Parallelization ($p$)**: 4 threads

### 2. Relational Schema Setup
The PostgreSQL datastore will contain a `users` table configured with the following schema:

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username VARCHAR(255) UNIQUE NOT NULL,
    hashed_password VARCHAR(255) NOT NULL,
    is_active BOOLEAN DEFAULT TRUE NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL
);
CREATE INDEX idx_users_username ON users (username);
```

### 3. Application Architecture Integration

To support this shift to stateful storage while maintaining optimal application performance, the system components will be refactored as follows:

```
+---------------------------------------------------------+
|                  FastAPI Delivery Layer                 | <--- [[entities/src-main]]
|   Exposes /api/v1/auth/login with Dependency Injection  |
+----------------------------+----------------------------+
                             |
                   Requests  | Yields DB Session
                 Connection  v
+----------------------------+----------------------------+
|             Database Session Context Provider           | <--- [[concepts/fastapi-dependency-injection]]
|       Manages async pgpool connections to PostgreSQL    |
+----------------------------+----------------------------+
                             |
                     Queries | Verifies Hash
                             v
+----------------------------+----------------------------+
|                Authentication Core Engine               | <--- [[entities/src-auth]]
|       Executes password verification via Argon2id      |
+---------------------------------------------------------+
```

#### A. Database Connection & Dependency Injection
We will introduce an asynchronous database connection pool using `SQLAlchemy` and `asyncpg`. This pool will be exposed to the FastAPI delivery layer (`[[entities/src-main]]`) using the framework's dependency injection container, aligning with the principles in `[[concepts/fastapi-dependency-injection]]`.

```python
# Proposed addition to a database module / injected into endpoints
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker

DATABASE_URL = settings.DATABASE_URL # Managed via [[concepts/configuration-management]]
engine = create_async_engine(DATABASE_URL, echo=False, future=True)
AsyncSessionLocal = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

async def get_db_session():
    async with AsyncSessionLocal() as session:
        yield session
```

#### B. Authentication Engine Refactoring (`[[entities/src-auth]]`)
The static validation logic in `[[entities/src-auth]]` will be replaced with a database-backed execution sequence. Below is the technical implementation path for generating and verifying hashes:

```python
from argon2 import PasswordHasher
from argon2.exceptions import VerifyMismatchError

# Initialize the PasswordHasher with secure defaults
ph = PasswordHasher(time_cost=3, memory_cost=65536, parallelism=4)

def hash_password(password: str) -> str:
    """Generates a secure Argon2id hash for a plaintext password."""
    return ph.hash(password)

def verify_password(hashed_password: str, password: str) -> bool:
    """Verifies a plaintext password against an Argon2id hash."""
    try:
        return ph.verify(hashed_password, password)
    except VerifyMismatchError:
        return False
```

#### C. Database-Backed Authentication Flow
The sequential authorization flow will adapt dynamically as detailed in `[[concepts/authentication-flows]]`. During login:
1. The client sends credentials via `OAuth2PasswordRequestForm` (`[[decisions/oauth2-form-integration]]`).
2. The FastAPI endpoint intercepts the request, retrieves a database session from `[[concepts/fastapi-dependency-injection]]`, and queries the database for the user record matching the given username.
3. If the user is found and `verify_password(user.hashed_password, password)` returns `True`, authentication succeeds.
4. The service generates a JWT using symmetric signing (`[[decisions/symmetric-vs-asymmetric-signing]]`), implementing stateless sessions (`[[concepts/stateless-sessions]]`).

---

## Consequences

### Positive
* **Cryptographic Strength**: Upgrading to Argon2id provides state-of-the-art protection against hardware-accelerated password recovery attacks.
* **12-Factor Alignment**: By pulling connection parameters from the updated Settings module (`[[concepts/configuration-management]]`), secrets and credentials are fully externalized.
* **Operational Readiness**: A relational database backing allows for standard identity management operations (such as registration, account locking, and role mapping).

### Negative / Risks
* **Computational Overhead**: Argon2id is intentionally resource-intensive. Under heavy login volume, password hashing will increase CPU and memory utilization on the application instances. This risk is mitigated by offloading token verification to stateless signature checks (`[[concepts/stateless-sessions]]` / `[[concepts/jwt-security]]`), ensuring database queries and hashing operations are strictly limited to the login path (`/api/v1/auth/login`).
* **Operational Complexity**: Introducing a PostgreSQL instance adds state to the system architecture, requiring connection pooling, network security rules (VPC confinement), regular backups, and schema migration management (via Alembic).