# ADR-0013: Pluggable Authentication — DB Today, ZTIAP Later

**Status:** Accepted

## Context

The product roadmap is explicit:

- **Today (demo / budget approval):** we need working auth *now*, against our own database. No ZTIAP integration is available yet.
- **After budget approval:** we move to Rancher + **ZTIAP** (zero-trust identity and access platform).

We must ship something real now without painting ourselves into a corner that ZTIAP integration will require rewriting. Options:

1. **Hardcode DB auth.** Ship fast, rewrite later. Rewrite risks breaking existing user sessions and API contracts.
2. **Integrate a full OIDC/SAML framework now and mock the IdP.** Overbuilt; the mock will diverge from ZTIAP's actual behavior.
3. **Define an interface, implement DB auth against it, swap implementations when ZTIAP lands.** Modest upfront cost, near-zero migration cost.

## Decision

Define an `AuthProvider` interface. Every API endpoint calls through it. Implementations:

- **`DbAuthProvider` (v1):** email + password, **Argon2id** hashing via `argon2-cffi`, signed session tokens (via `itsdangerous`) stored as `HttpOnly; Secure; SameSite=Lax` cookies. Server-side session records in Redis keyed by token, with configurable TTL and rotation on privilege change. CSRF token on state-changing endpoints.
- **`ZtiapAuthProvider` (v2):** swaps the implementation. API layer calls the same methods. Session storage in Redis still applies as an introspection cache in front of ZTIAP.

```python
class UserPrincipal(BaseModel):
    user_id: str
    email:   str
    roles:   list[str]
    claims:  dict

class AuthProvider(ABC):
    @abstractmethod
    async def login(self, credentials: dict) -> UserPrincipal: ...
    @abstractmethod
    async def verify_session(self, session_token: str) -> UserPrincipal: ...
    @abstractmethod
    async def logout(self, session_token: str) -> None: ...
```

Provider selection is a single config knob: `auth.provider = db | ztiap`.

## Consequences

**Positive**

- We ship real auth for the demo without blocking on ZTIAP.
- ZTIAP integration is a single-component change + config flip — no API, UI, or data-model refactor.
- `UserPrincipal` is the shape API code and authorization checks depend on; it's provider-agnostic.

**Negative**

- `DbAuthProvider` is a real identity provider — password policy, lockout, rotation all need real attention. **Cannot be shortcutted** even though it's "temporary."
- Two code paths for auth exist eventually; both must be kept test-covered.

**Discipline required**

- API code **never** reaches into `DbAuthProvider` specifics (e.g., password fields) — only the interface surface. A CI check flags imports of concrete providers outside `auth/`.
- When ZTIAP lands, keep `DbAuthProvider` for local-dev and automated tests so CI doesn't need a ZTIAP instance.
