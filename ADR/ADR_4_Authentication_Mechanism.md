# ADR 4: Use Token-Based Stateless Authentication

## Status
Accepted

## Context
Sismics Reader must authenticate users across multiple client types: a web SPA (AngularJS), an Android mobile app, and a desktop agent. The authentication mechanism must support stateless API access, "remember me" functionality for long-lived sessions, and optional header-based authentication for API proxy deployments.

Options considered:
- **HTTP Session-based authentication** — Traditional server-side sessions stored in memory or a session store.
- **JWT (JSON Web Tokens)** — Self-contained, signed tokens with embedded claims.
- **Custom token-based authentication** — Application-managed tokens stored in the database.
- **OAuth 2.0** — Delegated authorization framework, suitable for third-party integrations.

## Decision
We will use a **custom token-based stateless authentication** mechanism. Upon successful login (`POST /api/user/login`), the server generates a long-lived authentication token, stores it in the `T_AUTHENTICATION_TOKEN` database table, and returns it to the client. The client includes this token in subsequent requests via the `auth_token` query parameter or the `X-Auth-Token` HTTP header.

Two servlet filters implement the authentication pipeline:
1. **TokenBasedSecurityFilter** — Validates tokens against the database and sets the `UserPrincipal` in the security context.
2. **HeaderBasedSecurityFilter** — Optional alternative that reads user identity from HTTP headers (for reverse proxy scenarios).

Passwords are hashed using **jBCrypt** (bcrypt algorithm with CPU-intensive salt computation).

## Consequences

### Positive
- **Stateless API**: No server-side session state; tokens are validated per-request, enabling horizontal scalability.
- **Multi-client support**: The same token mechanism works for web, mobile, and desktop clients without client-specific authentication flows.
- **Long-lived sessions**: Tokens can persist across browser restarts and app relaunches, improving user experience with "remember me" functionality.
- **Secure password storage**: bcrypt hashing with salt prevents rainbow table attacks and resists brute-force attempts.
- **Flexible authentication**: Header-based auth support enables deployment behind reverse proxies with SSO integration.

### Negative
- **No token expiration by default**: Long-lived tokens without automatic expiration increase the risk of token theft and session hijacking.
- **Database lookup per request**: Each request requires a database query to validate the token, adding latency compared to self-contained tokens (like JWT).
- **No token refresh mechanism**: Clients cannot refresh tokens without re-authenticating, unlike OAuth 2.0 refresh token flows.
- **Custom implementation**: A custom auth system requires more careful security review compared to using established libraries or frameworks (e.g., Spring Security).
