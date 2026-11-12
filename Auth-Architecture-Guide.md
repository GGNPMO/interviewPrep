# Authentication & Authorization - Complete Architecture Guide
### Explained by a Principal Architect to a Fresher
**Stack:** ASP.NET Core MVC (.NET 8+), React, Web APIs, Microservices, SQL Server, OIDC Identity Provider

---

## How to read this document

Every topic follows the **same template**, so you always know where to look:

| Section | What it answers |
|---|---|
| **What is it?** | Plain-English definition |
| **Real-life analogy** | So it sticks in memory |
| **Why do we use it in our project?** | The business/technical reason |
| **How it works / where it appears** | Flow and code |
| **Advantages** | What we gain |
| **Disadvantages** | What it costs us (every choice has a cost) |
| **Alternatives** | What else we could have used and when |
| **Modern / AI angle** | How newer tech or AI changes or helps this |

> **Principal's rule #1:** There is no "best" technology. There is only the *right trade-off for the context*. Always be able to say: *why this, why not that, and what is the risk.*

---

# TABLE OF CONTENTS

**PART 1 - Foundations (start here)**
1.1 Authentication vs Authorization · 1.2 Core vocabulary · 1.3 Hashing vs Encryption vs Signing · 1.4 HTTPS/TLS · 1.5 Stateful vs Stateless · 1.6 The big picture of our project

**PART 2 - Authentication Mechanisms**
2.17 Account lifecycle and recovery
2.1 Username + Password · 2.2 Password hashing · 2.3 Cookie authentication · 2.4 ASP.NET Core Identity · 2.5 JWT · 2.6 OAuth 2.0 · 2.7 OpenID Connect (OIDC) · 2.8 SAML · 2.9 Windows / Kerberos / NTLM · 2.10 API Keys · 2.11 Basic Auth · 2.12 Client certificates / mTLS · 2.13 MFA · 2.14 Passkeys / WebAuthn · 2.15 Social & Enterprise login (federation) · 2.16 Magic link / OTP / passwordless

**PART 3 - Authorization Models**
3.11 End-to-end authorization example
3.1 Roles (RBAC) · 3.2 Claims-based · 3.3 Policy-based · 3.4 Resource-based · 3.5 ABAC · 3.6 ACL · 3.7 Scopes · 3.8 Externalised policy engines · 3.9 Multi-tenancy · 3.10 Relationship-based (ReBAC)

**PART 4 - Authentication in the MVC App (internals)**
4.1 Request pipeline · 4.2 Schemes, handlers · 4.3 Challenge / Forbid · 4.4 `[Authorize]`, `[AllowAnonymous]`, fallback policy · 4.5 Login lifecycle step by step · 4.6 Logout · 4.7 Session vs TempData

**PART 5 - React (SPA) Authentication**
5.1 Why SPAs are different · 5.2 Token storage options · 5.3 Auth Code + PKCE · 5.4 BFF pattern · 5.5 Route guards & UI authorization · 5.6 CORS

**PART 6 - Microservices Security**
6.1 API Gateway · 6.2 Token validation in each service · 6.3 Service-to-service: Client Credentials · 6.4 User context propagation / OBO / Token Exchange · 6.5 mTLS & service mesh · 6.6 Messaging security · 6.7 Secrets management · 6.8 Workload / managed identity · 6.9 Sync vs async communication (security view)

**PART 7 - Attacks & Protections**
7.1 OWASP Top 10 overview · 7.2 CSRF · 7.3 XSS · 7.4 SQL Injection · 7.5 Brute force & credential stuffing · 7.6 Session fixation / hijacking · 7.7 Open redirect · 7.8 Security headers & CSP · 7.9 Rate limiting · 7.10 Data Protection API · 7.11 Insecure Direct Object Reference (IDOR) · 7.12 Confused deputy

**PART 8 - Operations & Governance**
8.1 Token lifetimes & refresh · 8.2 Key rotation · 8.3 Revocation · 8.4 Logging & auditing · 8.5 Monitoring & alerting · 8.6 Testing security · 8.7 Compliance (GDPR, PCI, SOC2, HIPAA, IRDAI-type) · 8.8 Threat modelling

**PART 9 - Modern Trends & AI**
9.1 AI in authentication (risk-based, anomaly detection) · 9.2 Passkeys & passwordless · 9.3 Zero Trust · 9.4 Authenticating AI agents / LLM tools · 9.5 AI-generated code security risks · 9.6 Deepfakes & AI-driven attacks · 9.7 Policy as code

**PART 10 - Putting It Together**
10.1 Reference architecture for our project · 10.2 Decision guide · 10.3 Common mistakes · 10.4 Checklist · 10.5 Glossary

---

# PART 1 - FOUNDATIONS

## 1.1 Authentication vs Authorization

### What is it?
- **Authentication (AuthN)** = *"Who are you?"* - proving identity.
- **Authorization (AuthZ)** = *"What are you allowed to do?"* - checking permission.

### Real-life analogy
At an **airport**:
- Showing your **passport** at check-in = authentication (they confirm you are who you claim).
- Your **boarding pass** says which flight, which seat, whether you can enter the lounge = authorization.
You can be authenticated (valid passport) but *not authorized* (wrong gate).

### Why it matters in our project
Our claims system has customers, adjusters, managers and admins. A customer must see **only their own claims**; an adjuster must see **assigned claims**; only a manager can **approve payouts**. Without AuthZ, anyone who logs in can see or do everything.

### HTTP status codes (very commonly asked)
| Code | Meaning | When |
|---|---|---|
| **401 Unauthorized** | Not authenticated (really "unauthenticated") | No/invalid cookie or token |
| **403 Forbidden** | Authenticated but not allowed | User lacks role/policy |
| **404 Not Found** | Sometimes used deliberately to hide that a resource exists | Prevent information leak |

### Order of operations
`Authentication → Authorization → Business logic`. Never the other way round.

---

## 1.2 Core vocabulary (learn these first)

| Term | Meaning | Simple example |
|---|---|---|
| **Identity** | A representation of a user/service | "Priya, employee id 4521" |
| **Principal** | The authenticated entity in .NET: `ClaimsPrincipal` | `HttpContext.User` |
| **Claim** | A fact about the user: key + value | `email=priya@x.com`, `role=Adjuster` |
| **Credential** | Secret used to prove identity | Password, certificate, OTP |
| **Token** | A piece of data that represents a login/permission | JWT access token |
| **Session** | Server remembers you between requests | Cookie holding session/ticket |
| **Identity Provider (IdP)** | System that logs users in and issues tokens | Entra ID, Keycloak, Okta |
| **Relying Party / Client** | App that trusts the IdP | Our MVC app |
| **Resource Server** | API that accepts tokens | Orders API |
| **Scope** | Permission an app asks for on behalf of a user | `orders.read` |
| **Audience (aud)** | Who the token is meant for | `orders-api` |
| **Issuer (iss)** | Who created the token | `https://login.company.com` |
| **Subject (sub)** | Unique user id in a token | `a1b2-...` |
| **SSO** | Log in once, access many apps | Corporate login |
| **Federation** | Trusting another organisation's IdP | "Sign in with Google" |

---

## 1.3 Hashing vs Encryption vs Signing

| | Purpose | Reversible? | Key? | Example in our project |
|---|---|---|---|---|
| **Hashing** | Store passwords, verify integrity | **No** (one-way) | Salt (not secret key) | Password stored as PBKDF2/Argon2 hash |
| **Encryption** | Hide data, only key-holder can read | **Yes** | Symmetric (AES) or asymmetric (RSA) | Auth cookie is encrypted by Data Protection; TLS |
| **Signing** | Prove data wasn't changed and who made it | Verify only | Private key signs, public key verifies | JWT signature (RS256) |

**Analogy:** Hashing = shredding a document and keeping only the "fingerprint of the shredded pile". Encryption = locking it in a box. Signing = sealing the box with your personal wax stamp.

**Common mistake for freshers:** "JWT is encrypted." **No - a normal JWT is only signed.** Anyone can Base64-decode and *read* it. Never put secrets in a JWT.

---

## 1.4 HTTPS / TLS

### What is it?
TLS encrypts traffic between browser and server (HTTPS) so nobody on the network can read or alter it.

### Why in our project?
Cookies, tokens and passwords travel in HTTP requests. Without HTTPS they can be stolen on public Wi-Fi (man-in-the-middle).

### How we use it
- HTTPS everywhere, **HSTS** header (browser always uses HTTPS), TLS 1.2+ only.
- Certificates from a CA (Let's Encrypt, DigiCert, internal PKI); auto-renewed.
- TLS termination at the load balancer/ingress; internal traffic can be re-encrypted (mTLS).

| Advantages | Disadvantages |
|---|---|
| Confidentiality + integrity + server identity | Certificate management/renewal overhead |
| Required by modern browsers for `Secure` cookies, HTTP/2, PWAs | Slight CPU/latency overhead (negligible today) |

**Alternatives:** None for internet traffic. (VPN/IPsec is for network-level, not a replacement.)
**Modern angle:** Automated cert issuance (ACME), short-lived certs, TLS 1.3, post-quantum hybrid key exchange emerging.

---

## 1.5 Stateful vs Stateless authentication

| | Stateful (session) | Stateless (token) |
|---|---|---|
| Server stores | Session data / ticket | Nothing (token carries data) |
| Revocation | Easy (delete session) | Hard (valid until expiry) |
| Scaling | Needs shared store (Redis) | Easier horizontally |
| Typical | MVC cookie, BFF | API JWT |

**Principal's note:** Most real systems are **hybrid** - stateful cookie at the browser edge, stateless tokens between services.

---

## 1.6 The big picture of our project

```
 [Browser: MVC pages + React widgets]
        |  (HTTPS, cookie)
        v
 [MVC Web App / BFF]  -----OIDC------>  [Identity Provider]  (users, MFA, SSO)
        |  (Bearer access token)
        v
 [API Gateway (YARP)]  -->  [Claims API] [Policy API] [Payment API] ...
                                  |  gRPC / REST (service token)      |  events (broker)
                                  v                                    v
                             [SQL Server]                    [RabbitMQ / Service Bus]
```

Layers of security (**defence in depth**): TLS → IdP login + MFA → cookie/session → token validation → authorization policies → data-level checks → audit logs. If one layer fails, the next still protects.

---

# PART 2 - AUTHENTICATION MECHANISMS

## 2.1 Username + Password (Forms authentication)

### What is it?
User enters an id and secret; server verifies against stored data.

### Analogy
A house key: whoever holds it gets in - no questions asked.

### Why in our project?
Simplest starting point; needed for internal/local accounts when no external IdP is present.

### How it works
1. User posts credentials over HTTPS.
2. Server finds user, hashes the input with stored salt, compares with stored hash.
3. On success → issue cookie/token.

| Advantages | Disadvantages |
|---|---|
| Universally understood, easy to build | Users reuse/forget/choose weak passwords |
| No third-party dependency | Phishing, brute force, credential stuffing |
| Works offline/on-prem | You carry the **liability** for storing credentials |
| | Support cost (resets, lockouts) |

**Alternatives:** SSO/OIDC, passkeys, magic links, social login.
**Modern angle:** Industry is moving to **passwordless** (passkeys). Follow NIST 800-63B: favour length over complexity, check against breached-password lists, don't force periodic rotation.

---

## 2.2 Password hashing

### What is it?
Converting a password into a fixed-size irreversible string using a slow algorithm plus a random **salt** (unique per user).

### Why?
If the DB leaks, attackers should not get passwords. Plain text or fast hashes (MD5/SHA1/SHA256) are crackable in seconds.

### In .NET
ASP.NET Core Identity's `PasswordHasher<TUser>` uses **PBKDF2 (HMAC-SHA512, many iterations) + salt**. Format versioned so it can be upgraded transparently on next login.

| Algorithm | Notes |
|---|---|
| **PBKDF2** | Built-in, FIPS-friendly, good default |
| **bcrypt** | Mature, memory-light |
| **scrypt** | Memory-hard |
| **Argon2id** | Current recommendation (memory-hard, resists GPU cracking) |
| MD5/SHA-1/SHA-256 plain | **Never** |

**Advantages:** DB leak ≠ immediate password leak.
**Disadvantages:** Slow by design (CPU cost, DoS risk on login endpoint → rate limit it).
**Extra:** Add a **pepper** (secret stored in Key Vault, not DB) for additional safety.

---

## 2.3 Cookie authentication

### What is it?
After login the server sends a small `Set-Cookie`. The browser automatically attaches it to every later request. The cookie holds an **encrypted + signed ticket** (user's claims) or an id pointing to server-side data.

### Analogy
A **wristband at a festival**: show once at the gate, and every stall just looks at your wrist.

### Why in our project?
Our MVC pages are server-rendered and the browser is the client. Cookies are the **most secure and natural** fit: the browser handles them, JavaScript can't read them (`HttpOnly`).

### How it works in ASP.NET Core
```csharp
builder.Services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
    .AddCookie(o =>
    {
        o.Cookie.Name = "__Host-app";
        o.Cookie.HttpOnly = true;                       // JS cannot read it
        o.Cookie.SecurePolicy = CookieSecurePolicy.Always; // HTTPS only
        o.Cookie.SameSite = SameSiteMode.Lax;           // limits CSRF
        o.ExpireTimeSpan = TimeSpan.FromMinutes(30);
        o.SlidingExpiration = true;                     // extend while active
        o.LoginPath = "/Account/Login";
        o.AccessDeniedPath = "/Account/Denied";
    });
```

| Advantages | Disadvantages |
|---|---|
| Browser handles it automatically | **CSRF risk** (cookie auto-sent) → need antiforgery |
| `HttpOnly` blocks XSS token theft | Not ideal for mobile / non-browser clients |
| Easy server-side revocation (with ticket store) | Needs shared Data Protection keys when scaled out |
| Mature and well supported | Size limit ~4 KB; big claim sets break it |
| | Cross-domain/third-party cookie restrictions |

**Alternatives:** JWT in header, server-side session id, reference tokens.
**Modern angle:** Device-bound session credentials (DBSC) and token binding research aim to stop cookie theft by malware ("pass-the-cookie" attacks); phishing-resistant MFA reduces account takeover but stolen session cookies are still a risk → short sessions + risk checks.

---

## 2.4 ASP.NET Core Identity

### What is it?
Microsoft's built-in membership library: user store, password hashing, roles, claims, lockout, 2FA, email confirmation, external logins, token providers.

### Why in our project?
Gives a **ready, tested** implementation for local accounts rather than writing risky custom login code.

### Key classes
`UserManager<T>`, `SignInManager<T>`, `RoleManager<T>`, `IdentityUser`, EF Core stores.

### Typical use
```csharp
builder.Services.AddIdentity<AppUser, IdentityRole>(o =>
{
    o.Password.RequiredLength = 12;
    o.Lockout.MaxFailedAccessAttempts = 5;
    o.Lockout.DefaultLockoutTimeSpan = TimeSpan.FromMinutes(15);
    o.SignIn.RequireConfirmedEmail = true;
}).AddEntityFrameworkStores<AppDbContext>().AddDefaultTokenProviders();
```

| Advantages | Disadvantages |
|---|---|
| Free, integrated, battle-tested | You operate user DB, resets, breach responsibility |
| Lockout, 2FA, confirmation built-in | Not an SSO server (can't issue OIDC tokens by itself) |
| Extensible (custom stores, claims) | Each app has separate user DB → silos |
| | Limited advanced features (passkeys need .NET 10-era support/extra libs, adaptive risk) |

**Alternatives:** Microsoft Entra ID / External ID, Keycloak, Duende IdentityServer, Auth0, Okta, AWS Cognito, Firebase Auth.
**When to choose which:**
- Small single app → ASP.NET Core Identity.
- Many apps / enterprise SSO / compliance → dedicated IdP.
**Modern angle:** Managed IdPs now bundle adaptive/risk-based auth, bot detection and passkeys - features that are costly to build yourself.

---

## 2.5 JWT (JSON Web Token)

### What is it?
A compact string `header.payload.signature` (each part Base64URL). Payload has claims (`sub`, `iss`, `aud`, `exp`, `role`...). The signature proves it was issued by a trusted party and not altered.

### Analogy
A **signed concert ticket** - the gate can verify the stamp without calling the box office.

### Why in our project?
Our **APIs and microservices** validate tokens locally (no DB call) using the IdP's public keys (JWKS). Perfect for stateless, scalable services.

### Example payload
```json
{ "iss":"https://login.company.com", "sub":"a1b2", "aud":"claims-api",
  "exp":1731200000, "scope":"claims.read", "role":"Adjuster", "tid":"tenant-7" }
```

### Validation checklist (every API must do all)
Signature ✔ · `iss` ✔ · `aud` ✔ · `exp`/`nbf` ✔ · algorithm allow-list (reject `alg:none`) ✔ · scope/role ✔

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
  .AddJwtBearer(o => {
      o.Authority = "https://login.company.com";   // downloads signing keys automatically
      o.Audience  = "claims-api";
      o.MapInboundClaims = false;
  });
```

| Advantages | Disadvantages |
|---|---|
| Stateless, fast, scalable | **Hard to revoke** before expiry |
| Works across domains, mobile, services | Readable by anyone (not encrypted) |
| Standard, many libraries | Bigger than a session id; claims go stale |
| Carries claims (fewer DB lookups) | Misconfiguration is common (aud, alg, long expiry) |
| | Storing in browser `localStorage` = XSS risk |

**Alternatives:**
- **Opaque/reference tokens** + introspection (revocable, but extra call).
- **PASETO** (safer defaults than JWT).
- **Session cookies** for browsers.
- **Macaroons/biscuit** (attenuable tokens, niche).
**Modern angle:** **DPoP** (proof-of-possession) and mTLS-bound tokens make stolen tokens useless to attackers; short lifetimes + refresh rotation is standard.

---

## 2.6 OAuth 2.0

### What is it?
A **delegation/authorization framework**: lets an app get limited access to a resource **without sharing the user's password**. It is about *access*, not login.

### Analogy
**Valet key** for a car: opens the door and starts the engine, but not the boot or glovebox. You gave the valet limited power, not your master key.

### Main roles
Resource Owner (user) · Client (app) · Authorization Server (IdP) · Resource Server (API).

### Grant types (flows) - know when to use each
| Flow | Use case | Status |
|---|---|---|
| **Authorization Code + PKCE** | Web apps, SPAs, mobile (user present) | ✅ **Recommended** |
| **Client Credentials** | Service-to-service, no user | ✅ |
| **Refresh Token** | Get new access token silently | ✅ (rotate it) |
| **Device Code** | TVs / CLI tools without browser | ✅ |
| **Token Exchange / OBO** | Service acts on behalf of user | ✅ |
| Implicit | Old SPAs | ❌ Deprecated |
| Resource Owner Password | App collects password directly | ❌ Deprecated |

| Advantages | Disadvantages |
|---|---|
| Users never share passwords with apps | Complex; many ways to misconfigure |
| Fine-grained scopes, revocable | Not authentication by itself (that's OIDC) |
| Industry standard, huge ecosystem | Multiple RFCs to follow (OAuth 2.1 consolidates best practice) |

**Alternatives:** SAML (older, enterprise), API keys (simple, coarse), mTLS.
**Modern angle:** OAuth 2.1 (PKCE mandatory, no implicit), **Rich Authorization Requests (RAR)**, **DPoP**, and OAuth is now also the basis for authorizing AI agents/tools (see 9.4).

---

## 2.7 OpenID Connect (OIDC)

### What is it?
A thin **identity layer on top of OAuth 2.0**. Adds an **ID token** (who logged in) and a `/userinfo` endpoint. OAuth = "can this app access X?", OIDC = "who is this user?".

### Why in our project?
Our MVC app uses OIDC to **delegate login** to the company IdP: gets SSO, MFA, password policy, audit - for free and consistently across all apps.

### Key endpoints (from discovery document `/.well-known/openid-configuration`)
`/authorize` · `/token` · `/userinfo` · `/jwks` · `/end_session`

### Tokens you will hear about
| Token | For | Audience |
|---|---|---|
| **ID token** | The client app to learn who signed in | The client |
| **Access token** | Calling APIs | The API |
| **Refresh token** | Getting new access tokens | The IdP |

### Flow (Authorization Code + PKCE)
1. App redirects browser to IdP with `code_challenge`, `state`, `nonce`.
2. User authenticates at IdP (password + MFA).
3. IdP redirects back with a one-time `code`.
4. App's **server** exchanges `code` + `code_verifier` for tokens (back-channel).
5. App validates ID token, creates its own cookie session.

**Why `state`, `nonce`, PKCE?** `state` = anti-CSRF on callback. `nonce` = ties ID token to this login (replay protection). PKCE = stops stolen authorization codes being used.

| Advantages | Disadvantages |
|---|---|
| Central login, SSO, MFA, one user directory | IdP is a **critical dependency** (outage = nobody logs in) |
| Apps never see passwords | Setup complexity (redirect URIs, clock skew, claims mapping) |
| Standard across vendors | Vendor lock-in risk if using proprietary extensions |
| Easy to add new apps | Debugging redirects is painful |

**Alternatives:** SAML 2.0 (legacy enterprise), WS-Federation (legacy), local Identity.
**Modern angle:** OpenID Federation, FAPI profiles (financial-grade), verifiable credentials/decentralised IDs (emerging), Shared Signals/CAEP (IdP pushes "session revoked/risk changed" events to apps in near-real-time).

---

## 2.8 SAML 2.0

### What is it?
XML-based SSO protocol where IdP sends a signed **assertion** about the user to the Service Provider. Common in older enterprise/government/SaaS integrations.

### Why in our project?
Only if a corporate customer insists on SAML (many B2B customers do). Otherwise prefer OIDC.

| Advantages | Disadvantages |
|---|---|
| Mature, widely supported by enterprises | XML, verbose, heavy; many past security bugs (XML signature wrapping) |
| Rich attribute exchange | Poor fit for SPAs/mobile/APIs |
| | Harder to debug than OIDC |

**Alternatives:** OIDC (modern default). A broker (Keycloak/Entra) can **translate SAML ↔ OIDC** so our app only speaks OIDC.

---

## 2.9 Windows / Kerberos / NTLM (Integrated authentication)

### What is it?
Domain-joined users get silent login using Active Directory tickets.

### Why in our project?
Internal intranet tools used only by employees on corporate network.

| Advantages | Disadvantages |
|---|---|
| Seamless SSO inside the domain | Only works inside/with the domain; not internet-friendly |
| No password prompt | Tied to Windows/IIS/AD (Linux needs extra setup) |
| | NTLM is legacy and weak |

**Alternatives:** OIDC with Entra ID (works on any device, with MFA/conditional access).
**Modern angle:** Hybrid identity (AD synced to Entra) is the usual migration path.

---

## 2.10 API Keys

### What is it?
A long random string sent in a header (`X-API-Key`) identifying the **calling application**, not a user.

### Why in our project?
Simple integrations with partners/third-party systems or webhooks where OAuth is overkill.

| Advantages | Disadvantages |
|---|---|
| Very simple | Identifies the app only, not the user |
| Easy for partners | If leaked, valid until revoked; often long-lived |
| | No standard scopes/expiry; must be hashed in DB, rotated |

**Good practice:** Store only a **hash** of the key; allow multiple keys per client for rotation; per-key rate limit and scope; never put in URLs (logged everywhere).
**Alternatives:** OAuth2 client credentials (preferred), HMAC-signed requests (webhooks), mTLS.

---

## 2.11 Basic Authentication

### What is it?
`Authorization: Basic base64(user:password)` sent on **every** request.

### Where?
Only quick internal tools or legacy integrations - and **only over HTTPS**.

| Advantages | Disadvantages |
|---|---|
| Trivial to implement | Credentials sent every time; no logout; no MFA |
| | Base64 is **not encryption** |

**Alternatives:** OAuth2 tokens, cookies, API keys.

---

## 2.12 Client Certificates / mTLS

### What is it?
In normal TLS only the server shows a certificate. In **mutual TLS** the client also presents one, proving *its* identity.

### Why in our project?
Strong **service identity** between microservices and with high-security partners (banks, regulators).

| Advantages | Disadvantages |
|---|---|
| Very strong; not phishable; encrypts + authenticates | Certificate issuance, rotation, revocation overhead (PKI) |
| No shared secret in headers | Harder to debug; ops-heavy without automation |
| Good for zero-trust | Doesn't tell you *which user*, only which workload |

**Alternatives:** JWT client credentials, managed identities.
**Modern angle:** **SPIFFE/SPIRE** and service meshes issue short-lived workload certs automatically (hours, not years).

---

## 2.13 Multi-Factor Authentication (MFA)

### What is it?
Combine **2+ factors**: something you **know** (password), **have** (phone/security key), **are** (fingerprint).

### Why in our project?
Passwords are the #1 breach vector. MFA blocks the vast majority of account-takeover attempts, and is typically **mandatory** for approvers, admins and payment actions.

### Methods (weak → strong)
| Method | Phishing-resistant? | Notes |
|---|---|---|
| SMS OTP | ❌ | SIM-swap risk; use only as fallback |
| Email OTP | ❌ | Depends on mailbox security |
| TOTP app (Authenticator) | ❌ (can be phished in real time) | Good baseline |
| Push approve | ❌ (MFA fatigue attacks) | Use **number matching** |
| FIDO2 security key / **Passkey** | ✅ | Best |

**Step-up authentication:** ask for MFA again only for sensitive actions (e.g. approve claim > ₹10 lakh). Implemented via `acr`/`amr` claims and a policy.

| Advantages | Disadvantages |
|---|---|
| Massive reduction in account takeover | Friction for users; support cost (lost device) |
| Meets compliance | SMS/push can be attacked (SIM swap, fatigue) |

**Alternatives/complements:** Risk-based/adaptive MFA (only prompt when risky), passkeys, device trust.
**Modern angle:** AI/ML risk engines decide *when* to challenge (new device, impossible travel, unusual behaviour) - see 9.1.

---

## 2.14 Passkeys / WebAuthn (FIDO2)

### What is it?
Public-key cryptography login. Device creates a key pair per site; private key never leaves the device (protected by biometric/PIN); server stores the public key.

### Analogy
A lock that only opens for **your fingerprint-protected key** that can't be copied or emailed - and that refuses to work on a fake lookalike website.

### Why consider it?
Eliminates passwords, phishing and credential stuffing; better UX.

| Advantages | Disadvantages |
|---|---|
| Phishing-resistant, no shared secret on server | Account recovery design is hard |
| Faster login, fewer helpdesk resets | Device/ecosystem sync differences; user education |
| Supported by major browsers/OS | Legacy browsers/kiosks need fallback |

**Alternatives:** Password + TOTP, hardware keys.
**In .NET:** Fido2NetLib, IdP-native (Entra, Keycloak, Auth0), and newer ASP.NET Core Identity versions add passkey support.

---

## 2.15 Social & Enterprise Federation

### What is it?
"Sign in with Google/Microsoft/Apple" or a customer's corporate IdP. Our IdP **brokers** them and gives our app a single, normalised token.

| Advantages | Disadvantages |
|---|---|
| Lower sign-up friction, no password to manage | Dependence on third party; their outages/policy changes |
| Better security (their MFA) | Account-linking pitfalls (email takeover) |
| | Privacy/compliance questions for data sharing |

**Pitfall:** Link accounts by `(issuer, subject)` - **never by email alone** (email can be re-used or unverified).

---

## 2.16 Magic link / OTP / passwordless email

### What is it?
User receives a one-time link or code by email/SMS and logs in with it.

| Advantages | Disadvantages |
|---|---|
| No password to remember | Relies on email/SMS security & delivery |
| Good for infrequent users (customers) | Link forwarding/prefetch by email scanners can burn single-use links |

**Good practice:** Single-use, short expiry (5-15 min), bind to browser/device, rate limit, generic error messages.

---

## 2.17 Account lifecycle and recovery

Authentication is more than the login screen. The account lifecycle must cover **provisioning, verification, recovery, suspension, deprovisioning and reactivation**.

### Recommended lifecycle

1. Create or provision the account with a stable identifier (`issuer + subject` for federated users, or an internal user id for local users).
2. Verify the email or invitation before granting normal access; do not treat an unverified email as proof of identity.
3. Assign the least-privileged default role. A user should not receive `Admin` merely because a client supplied that value during registration.
4. Require MFA or a passkey for privileged users and step-up authentication for high-risk actions.
5. Support recovery with short-lived, single-use tokens. Recovery must not silently weaken MFA for an administrator.
6. Disable access immediately when an employee leaves, a tenant is suspended, or compromise is suspected. Revoke sessions and refresh-token families.
7. Keep an audit record of invitations, verification, recovery, role changes, lockouts and deprovisioning.

### Example: safe password-reset flow

```text
POST /account/forgot-password
  -> Always return: "If the account exists, an email was sent."
  -> Store only a hash of a random reset token.
  -> Expire it after 15 minutes and accept it once.

POST /account/reset-password
  -> Validate token, password policy and rate limit.
  -> Change the password, revoke sessions/refresh tokens,
     and notify the user through a separate channel.
```

Never put reset tokens in logs, analytics URLs or support tickets. Do not reveal whether an email address exists; that turns recovery into a user-enumeration endpoint.

---

# PART 3 - AUTHORIZATION MODELS

> Authorization answers **"can THIS user do THIS action on THIS resource right now?"**

## 3.1 Role-Based Access Control (RBAC)

### What is it?
Users are given **roles** (Admin, Adjuster, Customer); permissions attach to roles.

### Analogy
Office badges: "Manager" badge opens the finance room, "Visitor" badge doesn't.

### In code
```csharp
[Authorize(Roles = "Admin,Manager")]
public IActionResult ApproveClaim(int id) { ... }
```

| Advantages | Disadvantages |
|---|---|
| Simple, easy to explain and audit | **Role explosion** ("Adjuster-North-Level2-ReadOnly...") |
| Native `[Authorize(Roles)]` support | Can't express data rules ("only own region") |
| Good for coarse access | Roles hard-coded in code → redeploy to change |

**Alternatives:** Claims/policy (3.2, 3.3), ABAC (3.5).
**Use when:** Few stable roles, coarse screens/menus.

---

## 3.2 Claims-based authorization

### What is it?
Decision uses **claims** in the user's identity (`department=Claims`, `region=West`, `level=3`).

```csharp
[Authorize(Policy = "WestRegionOnly")]
// policy: RequireClaim("region","West")
```

| Advantages | Disadvantages |
|---|---|
| More flexible than pure roles | Claims can go **stale** until token/cookie refresh |
| IdP can supply claims centrally | Too many claims → big cookie/token |
| | Trusting claim values requires trusting the issuer |

**Alternatives:** Fetch permissions live from a permission service; use short token lifetime.

---

## 3.3 Policy-based authorization

### What is it?
Named rules combining requirements (roles, claims, custom logic) managed in one place. **Recommended default in ASP.NET Core.**

```csharp
builder.Services.AddAuthorization(o =>
{
    o.AddPolicy("CanApprovePayout", p => p
        .RequireAuthenticatedUser()
        .RequireRole("Manager")
        // Many OAuth providers emit scope as one space-delimited claim.
        .RequireAssertion(c => c.User.FindAll("scope")
            .SelectMany(x => x.Value.Split(' ', StringSplitOptions.RemoveEmptyEntries))
            .Contains("payout.approve", StringComparer.Ordinal))
        .RequireAssertion(c => c.User.HasClaim("amr", "mfa")));   // MFA required

    o.FallbackPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser().Build();                      // secure by default
});
```

| Advantages | Disadvantages |
|---|---|
| Central, testable, reusable | Slightly more code than `[Authorize(Roles)]` |
| Composable, supports custom handlers & DI | Policies scattered across projects if not governed |
| Fallback policy = secure by default | |

**Fresher tip:** Name policies after the **business action** (`CanApprovePayout`), not after roles - roles change, actions don't.

---

## 3.4 Resource-based authorization

### What is it?
Decision depends on **the specific record**: "Can *this* adjuster edit *this* claim?" The record must be loaded first.

```csharp
var claim = await _repo.GetAsync(id);
var result = await _authService.AuthorizeAsync(User, claim, "EditClaim");
if (!result.Succeeded) return Forbid();
```
Handler checks `claim.AssignedTo == User.Id` or `claim.Region == userRegion`.

| Advantages | Disadvantages |
|---|---|
| Prevents **IDOR** (accessing others' data by changing an id) | Must be called in every action/handler (easy to forget) |
| Expresses real business rules | Extra DB read before decision |

**Best practice:** Also apply the same filter in the **query** (`WHERE region = @userRegion`) so unauthorised rows are never loaded.

---

## 3.5 Attribute-Based Access Control (ABAC)

### What is it?
Rules use attributes of **user + resource + action + environment** (time, IP, device). Example: "Adjusters can approve claims < ₹5 L, in own region, during working hours, from managed device."

| Advantages | Disadvantages |
|---|---|
| Very fine-grained and dynamic | Complex to design, test, audit |
| Fewer roles | Performance overhead if attributes fetched live |
| | "Why was I denied?" can be hard to explain |

**Alternatives:** RBAC (simpler), ReBAC (relationship-heavy).

---

## 3.6 ACL (Access Control Lists)

### What is it?
Each resource holds a list of who can do what (like file sharing: "Alice: read, Bob: edit").

| Advantages | Disadvantages |
|---|---|
| Very precise per-item control | Hard to manage at scale; storage and query cost |
| Natural for document/file sharing | Hard to answer "what can Alice access?" |

---

## 3.7 Scopes (OAuth permissions)

### What is it?
Coarse permissions on **tokens** that limit what a client app can do (`claims.read`, `claims.write`).

**Important:** Scope = what the *app* may do for the user. It does **not** replace user-level authorization. Effective permission = **scope ∩ user's actual rights**.

---

## 3.8 Externalised policy engines (Policy as Code)

### What is it?
A separate **Policy Decision Point (PDP)** answers "allow/deny" using rules written as code: **OPA (Rego)**, **Cerbos**, **Cedar (AWS)**, **OpenFGA / SpiceDB** (relationship-based), Azure-based permission services.
Services act as **Policy Enforcement Points (PEP)**: ask PDP, then enforce.

### Why use it?
When 40 microservices need the same rules, copy-pasting `if` statements is unmaintainable and un-auditable.

| Advantages | Disadvantages |
|---|---|
| Single source of truth, versioned in Git, testable | Extra infra + network hop (mitigate with sidecar/cache) |
| Non-developers can review policy | New language (Rego/Cedar) to learn |
| Easy audit of "who can do what" | Over-engineering for small systems |

**Modern angle:** AI assistants can help *draft and explain* policies and generate test cases, but humans must review - a wrongly generated policy is a security hole (see 9.5).

---

## 3.9 Multi-tenancy

### What is it?
One system serving many customers (tenants), data isolated.

**Authorization rules to enforce:**
- `tenant_id` claim in the token; **every query filtered by tenant** (EF global query filter).
- Validate token issuer/tenant; no cross-tenant joins.
- Per-tenant roles/config.

```csharp
modelBuilder.Entity<Claim>().HasQueryFilter(c => c.TenantId == _tenantProvider.TenantId);
```

| Advantages | Disadvantages |
|---|---|
| Cost-efficient SaaS | A single missing filter = data breach across tenants |
| | Noisy neighbour, per-tenant customization complexity |

**Alternatives:** DB-per-tenant (strong isolation, higher cost), schema-per-tenant, row-level security in SQL Server.

---

## 3.10 Relationship-Based Access Control (ReBAC)

### What is it?
Permissions derive from **relationships** in a graph (Google Zanzibar model): "user is *member of* team → team *owns* folder → folder *contains* doc".

**Tools:** OpenFGA, SpiceDB, Auth0 FGA.

| Advantages | Disadvantages |
|---|---|
| Natural for sharing/hierarchies | New mental model and infra |
| Scales to billions of relations | Overkill for simple role systems |

---

## 3.11 End-to-end authorization example

Use this sequence for every protected business operation:

`authenticate caller -> validate token -> check coarse permission -> load resource safely -> check resource and tenant -> perform action -> audit result`

### Example: customer reads one claim

Assume the access token contains `sub=user-42`, `tid=tenant-7`, and `scope=claims.read`. The endpoint must check all of the following:

1. The JWT signature, issuer, audience, expiry and not-before time are valid.
2. The caller has the `claims.read` scope.
3. The requested claim belongs to `tenant-7`.
4. The caller owns the claim, or an explicit policy grants access (for example, an assigned adjuster or manager).

```csharp
[Authorize(Policy = "ClaimsRead")]
[HttpGet("api/claims/{id}")]
public async Task<IActionResult> GetClaim(Guid id, CancellationToken ct)
{
    var userId = User.FindFirstValue("sub");
    var tenantId = User.FindFirstValue("tid");
    if (userId is null || tenantId is null)
        return Unauthorized();

    // Tenant and ownership constraints belong in the query, not only in the UI.
    var claim = await db.Claims.SingleOrDefaultAsync(c =>
        c.Id == id && c.TenantId == tenantId &&
        (c.CustomerId == userId || c.AssignedTo == userId), ct);

    // Returning 404 avoids confirming that another tenant's claim exists.
    if (claim is null)
        return NotFound();

    return Ok(claim);
}
```

The React menu may hide an **Approve** button for a customer, but that is only usability. The API must still return `403` when an authenticated customer calls the approval endpoint. For a missing or cross-tenant resource, `404` is often preferable because it does not disclose resource existence.

### Effective permission

For a user-driven API call, the effective permission is the intersection of every relevant boundary:

```text
effective access = valid identity
                ∩ token audience and scope
                ∩ tenant membership
                ∩ user role/claims
                ∩ resource ownership or relationship
                ∩ business conditions (status, amount, MFA, device, time)
```

A role, scope or UI check alone is never sufficient for sensitive data or state-changing operations.

---

# PART 4 - AUTHENTICATION IN THE MVC APP (INTERNALS)

## 4.1 The request pipeline

```csharp
app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseRouting();
app.UseAuthentication();   // 1) Build HttpContext.User from cookie/token
app.UseAuthorization();    // 2) Check [Authorize] rules
app.MapControllerRoute(...);
```
**Order matters.** If `UseAuthorization` runs before `UseAuthentication`, everyone is anonymous.

**Why a pipeline?** Each middleware does one job; it is flexible, testable and lets cross-cutting concerns (logging, security headers) plug in.

## 4.2 Schemes and handlers

- **Scheme** = a named way of authenticating ("Cookies", "Bearer", "OpenIdConnect").
- **Handler** = the code that implements three actions: **Authenticate**, **Challenge**, **Forbid** (+ SignIn/SignOut).
- You can register several schemes (cookie for browser pages, JWT for `/api`) and choose per endpoint:
```csharp
[Authorize(AuthenticationSchemes = JwtBearerDefaults.AuthenticationScheme)]
```
**Advantage:** one app can serve both humans and machines. **Disadvantage:** confusing default-scheme issues when more than one is registered.

## 4.3 Challenge vs Forbid
- **Challenge** → unauthenticated → redirect to login (cookie) / 401 (API) / redirect to IdP (OIDC).
- **Forbid** → authenticated but denied → 403 / Access Denied page.

## 4.4 `[Authorize]`, `[AllowAnonymous]`, Fallback policy
- `[Authorize]` on controller/action requires login.
- `[AllowAnonymous]` opens specific endpoints (login, health, static pages).
- **Fallback policy** requires authentication by default everywhere → new endpoints are safe automatically. *Principal's rule: secure by default; open by exception.*

## 4.5 Login lifecycle, step by step (explain this to a fresher)

1. User opens `/Claims/Index` (protected).
2. `UseAuthentication` finds no valid cookie → user is anonymous.
3. `[Authorize]` fails → **Challenge** → redirect to login (or to IdP with OIDC).
4. User authenticates (password + MFA).
5. App calls `SignInAsync` → builds `ClaimsPrincipal` → encrypts & signs into cookie.
6. Browser stores cookie; redirect back to original URL (`ReturnUrl` validated with `Url.IsLocalUrl`).
7. Every next request: cookie → decrypted → `HttpContext.User` populated.
8. Authorization policies evaluated per endpoint.
9. Cookie renewed if sliding; expires after inactivity/absolute time.

## 4.6 Logout
- Clear local cookie **and** (with OIDC) end IdP session (RP-initiated logout).
- Use **POST + antiforgery** (not GET).
- Optionally **back-channel logout** so other apps also end the session.
- Revoke refresh tokens.

## 4.7 Session vs TempData vs Cookie
| Item | Stored where | Use |
|---|---|---|
| Auth cookie | Browser (encrypted) | Login state |
| `Session` | Server (memory/Redis) keyed by cookie id | Temporary user data (avoid for auth) |
| `TempData` | Cookie or session | One-time messages across redirect |
Behind a load balancer: use **Redis-backed** session and **shared Data Protection keys**; avoid sticky sessions as a "fix".

---

# PART 5 - REACT (SPA) AUTHENTICATION

## 5.1 Why SPAs are different
React runs **entirely in the user's browser** - anybody can inspect its code and storage. So a SPA is a **public client**: it cannot keep a secret. Any token it holds is exposed to JavaScript, hence to XSS.

## 5.2 Where to store tokens?

| Option | XSS exposure | CSRF exposure | Verdict |
|---|---|---|---|
| `localStorage` / `sessionStorage` | **High** | None | ❌ Avoid for sensitive apps |
| In-memory variable | Medium (lost on refresh) | None | ⚠️ OK with silent renew |
| HttpOnly cookie (via BFF) | Low | Need CSRF defence | ✅ **Best for our project** |

## 5.3 Authorization Code + PKCE for SPA (public client)
Used with MSAL.js / oidc-client-ts when no BFF exists. Works but tokens live in browser; use short lifetimes + rotating refresh tokens.

## 5.4 BFF (Backend-for-Frontend) pattern

### What is it?
A small server-side app that sits next to React: **does the OIDC login, holds tokens server-side, gives the browser only an HttpOnly cookie,** and forwards API calls with the access token.

### Why in our project?
Compliance-sensitive (insurance/finance) → no tokens in browser. Also removes most CORS complexity (same site).

| Advantages | Disadvantages |
|---|---|
| Tokens never reach JavaScript | Extra component to run and scale |
| Simpler CORS, same-site cookie | Session state to manage (Redis) |
| Central place for CSRF, refresh, logging | Slightly more latency (extra hop) |

**Alternatives:** SPA-only PKCE (simpler infra, weaker XSS posture); Next.js/SSR server doing the same BFF job; token-handler pattern (Duende BFF, YARP).
**Modern angle:** IETF "OAuth for Browser-Based Apps" best-current-practice recommends BFF for high-security apps.

## 5.5 Route guards and UI authorization
```tsx
function RequireRole({ role, children }) {
  const { user } = useAuth();            // from /bff/user
  return user?.roles.includes(role) ? children : <Navigate to="/forbidden" />;
}
```
**Golden rule:** Hiding buttons is **UX only**. The **API must always enforce** the same rule. Attackers don't use your UI.

## 5.6 CORS (Cross-Origin Resource Sharing)

### What is it?
Browser rule: JS from `app.example.com` can't read responses from `api.example.com` unless the API allows it.

### Why do we care?
Misconfigured CORS (`AllowAnyOrigin` + credentials) lets malicious sites call your API with the victim's cookies.

```csharp
builder.Services.AddCors(o => o.AddPolicy("spa", p => p
    .WithOrigins("https://app.example.com")
    .AllowCredentials().AllowAnyHeader().WithMethods("GET","POST","PUT","DELETE")));
```
**Note:** CORS is a *browser* protection, not server authentication. curl/Postman ignore it.

---

# PART 6 - MICROSERVICES SECURITY

## 6.1 API Gateway

### What is it?
Single front door (YARP, Ocelot, Azure APIM, Kong, NGINX) routing client requests to services.

### Why in our project?
Central place for TLS termination, authentication, rate limiting, routing, logging, version routing.

| Advantages | Disadvantages |
|---|---|
| One place for cross-cutting security | Single point of failure/bottleneck if not scaled |
| Hides internal topology | Temptation to put business logic in it |
| Rate limiting, WAF integration | **False sense of security** if services trust it blindly |

**Principal's rule:** Gateway authenticates **and** each service still validates tokens - *defence in depth*, never "the gateway handled it."
**Alternatives:** Direct client → service with mesh; BFF per client type; cloud API Management.

---

## 6.2 Token validation inside each service
Each service: validate JWT signature (cached JWKS), `iss`, `aud` (must equal *this* service), `exp`, then check scope/policy. A **shared NuGet security library** keeps all teams consistent.

---

## 6.3 Service-to-service: Client Credentials flow

### What is it?
A service logs in **as itself** to the IdP using its own id + secret/certificate/managed identity, gets an access token, and calls another service.

### Why?
Background jobs, internal calls with no user involved.

```csharp
builder.Services.AddHttpClient<IPolicyClient, PolicyClient>(c => c.BaseAddress = new("https://policy.internal"))
    .AddHttpMessageHandler<ClientCredentialsHandler>()   // gets & caches token
    .AddStandardResilienceHandler();                     // retry/timeout/circuit-breaker
```
(Handler requests token with `grant_type=client_credentials&scope=policy.read`, caches until ~1 min before expiry, uses a lock to avoid thundering herd.)

| Advantages | Disadvantages |
|---|---|
| Standard, auditable, scoped | Secret management burden (rotate!) |
| Short-lived tokens | No user context |
| Works with any language | Token endpoint dependency (cache & resilience needed) |

**Alternatives:** Managed identity / workload identity (no secret), mTLS only, API keys (weaker).

---

## 6.4 Propagating user context

| Option | Description | Verdict |
|---|---|---|
| Forward user's token | Same token to next service | ⚠️ Audience/scope problems, replay risk |
| **Token Exchange / OBO (RFC 8693)** | Swap user token for a new one with `aud=next-service`, reduced scope, `act=caller` | ✅ **Best** |
| Service token + `X-User-Id` header | Next service trusts caller's claim | ⚠️ Confused-deputy risk |
| Internal signed context token | Gateway mints internal JWT | Only if you can operate it safely |

**Why OBO?** Next service can check **both** "which user" and "which service is calling".

---

## 6.5 mTLS and service mesh

- **mTLS:** both sides show certs → encrypted and workload-authenticated.
- **Service mesh (Istio/Linkerd/Consul):** sidecar proxies enforce mTLS, retries, telemetry, **authorization policies** ("only `orders` may call `inventory`").

| Advantages | Disadvantages |
|---|---|
| Zero-trust networking without app code | Operational complexity, learning curve |
| Auto-rotating short-lived certs | Resource overhead (sidecars) - ambient/eBPF modes reduce it |
| Uniform telemetry | Not needed for small systems |

**Alternatives:** App-level JWT + Kubernetes NetworkPolicies; Dapr; cloud-native (AWS App Mesh, Azure Service Mesh).

---

## 6.6 Securing messaging (RabbitMQ / Service Bus / Kafka)

- TLS in transit; per-service credentials with **least privilege** (publish-only / consume-only per queue/topic).
- Don't put secrets/PII in plain messages; encrypt sensitive fields.
- Carry **correlation id + user context** in headers; consumers must **re-authorize** business operations (don't assume "message arrived = allowed").
- Optionally **sign messages** to prove origin.
- Protect DLQs - they contain real data.

| Advantages (async) | Disadvantages |
|---|---|
| Decouples services, resilient to downtime | Harder to propagate and verify user identity |
| Buffering, retry | Eventual consistency; duplicates → idempotency needed |

---

## 6.7 Secrets management

### What is it?
Storing passwords, connection strings, client secrets, signing keys safely.

**Never:** in Git, `appsettings.json` in prod, Docker images, logs, chat.
**Use:** Azure Key Vault / AWS Secrets Manager / HashiCorp Vault / Kubernetes secrets with external secrets operator; **managed identity** to fetch them; rotation policy; separate per environment.

| Advantages | Disadvantages |
|---|---|
| Central, audited, rotatable | Another dependency; startup failure if vault unreachable (cache/retry) |
| Fine-grained access | Complexity for local dev (use user-secrets) |

**Modern angle:** Prefer **secretless** designs - workload identity federation, short-lived credentials - so there's nothing long-term to steal. Secret-scanning tools (GitHub secret scanning, gitleaks) in CI block accidental commits.

---

## 6.8 Workload / Managed identity

### What is it?
The platform (Azure, AWS, Kubernetes) gives your service an identity automatically; it gets tokens without any stored secret.

| Advantages | Disadvantages |
|---|---|
| No secrets to store/rotate | Tied to the cloud/platform |
| Auditable, least-privilege RBAC | Local development needs fallbacks |

---

## 6.9 Sync vs async communication (security view)

| | Sync (REST/gRPC) | Async (events/commands) |
|---|---|---|
| Identity | Token on each call - simple | Must carry/verify in message |
| Failure handling | Timeouts, circuit breaker | Retry, DLQ, idempotency |
| Typical security concern | Token propagation, mTLS | Message tampering, over-trusting consumers |
Rule of thumb: **ask synchronously, announce asynchronously.**

---

# PART 7 - ATTACKS & PROTECTIONS

## 7.1 OWASP Top 10 (know the names)
Broken Access Control · Cryptographic Failures · Injection · Insecure Design · Security Misconfiguration · Vulnerable/Outdated Components · Identification & Authentication Failures · Software/Data Integrity Failures · Logging & Monitoring Failures · SSRF. (There's also an OWASP **API Security Top 10**: BOLA/IDOR, broken auth, excessive data exposure, etc.)

---

## 7.2 CSRF (Cross-Site Request Forgery)
**What:** A malicious site makes the victim's browser send a request to our site; the browser attaches the cookie automatically.
**Analogy:** Someone slips a pre-filled bank form in front of you while you're logged in; you sign without noticing.
**Protection:**
- `[ValidateAntiForgeryToken]` / `AutoValidateAntiforgeryTokenAttribute` globally.
- `SameSite=Lax/Strict` cookies.
- For SPAs/BFF: custom required header (e.g. `X-CSRF: 1`) or double-submit token.
- Don't change state with GET.
**Not affected:** Pure bearer-token header APIs (not auto-sent).

## 7.3 XSS (Cross-Site Scripting)
**What:** Attacker injects JavaScript that runs in our page and steals data/tokens or acts as the user.
**Protection:** Razor encodes by default (avoid `Html.Raw`); React escapes by default (avoid `dangerouslySetInnerHTML`); sanitize rich text (DOMPurify/HtmlSanitizer); **CSP**; `HttpOnly` cookies; dependency scanning (supply-chain XSS).

## 7.4 SQL Injection
**What:** User input becomes part of a SQL command.
**Protection:** Parameterized queries / EF Core LINQ; never string-concatenate SQL; least-privilege DB account; stored procs with parameters.

## 7.5 Brute force & credential stuffing
**Protection:** Lockout/backoff, rate limiting by IP+account, CAPTCHA after failures, MFA, breached-password check (HIBP k-anonymity), bot detection, generic error ("invalid credentials" - don't reveal which part).

## 7.6 Session fixation / hijacking
**Protection:** New session id on login, `HttpOnly+Secure` cookies, short lifetime, re-auth for sensitive actions, device/risk signals, HTTPS only.

## 7.7 Open redirect
**What:** `/login?returnUrl=https://evil.com`.
**Protection:** `Url.IsLocalUrl(returnUrl)` or allow-list; OIDC redirect URIs registered exactly.

## 7.8 Security headers & CSP
```csharp
ctx.Response.Headers["Content-Security-Policy"] = "default-src 'self'; script-src 'self'; frame-ancestors 'none'";
ctx.Response.Headers["X-Content-Type-Options"] = "nosniff";
ctx.Response.Headers["Referrer-Policy"] = "strict-origin-when-cross-origin";
// + HSTS via app.UseHsts()
```

## 7.9 Rate limiting
Built-in `AddRateLimiter` (fixed/sliding/token-bucket/concurrency). Apply stricter limits on `/login`, `/token`, password reset, OTP. Also at gateway/WAF.

## 7.10 Data Protection API (ASP.NET Core)
Encrypts/decrypts cookies, antiforgery tokens, TempData using rotating keys.
**Production must:** persist keys in shared store (Redis/Blob/DB), protect keys at rest (Key Vault), set `SetApplicationName`, same config on all instances. Missing this = "random logouts" in multi-instance deployments.

## 7.11 IDOR / Broken Object Level Authorization (BOLA)
**What:** `GET /api/claims/123` → change to `124` and see someone else's claim.
**Protection:** Resource-based authorization + tenant/owner filter in query; use unguessable ids (GUIDs) as *defence in depth*, not as the control.
**This is the #1 API vulnerability - always test for it.**

## 7.12 Confused deputy
A privileged service is tricked into acting for an attacker with its own power. **Protection:** token exchange (user + service identity), audience restriction, least privilege, validate who is asking *and* on whose behalf.

---

# PART 8 - OPERATIONS & GOVERNANCE

## 8.1 Token lifetimes & refresh

| Token | Typical lifetime | Why |
|---|---|---|
| Access token | 5-15 min | Limits damage if stolen |
| Refresh token | Hours-days, **rotating** | Convenience without re-login |
| Cookie session | 20-60 min sliding + absolute cap 8-12 h | Balance UX/security |
| ID token | Minutes (used once at login) | |

**Refresh token rotation + reuse detection:** if an old refresh token is replayed, revoke the whole token family.
**Trade-off:** Shorter = safer, more refresh traffic and more IdP load; longer = easier UX, bigger breach window.

## 8.2 Key rotation
- Signing keys (JWT): publish new key in JWKS ahead, sign with new, retire old after max token lifetime.
- Data Protection keys: automatic rotation (default 90 days) with shared persistent store.
- Client secrets/certs: rotate on schedule and on staff change; support two active secrets for zero-downtime rotation.
- Services must **cache JWKS but refresh on unknown `kid`**.

## 8.3 Revocation - the hard problem
| Approach | Notes |
|---|---|
| Short-lived tokens | Simplest; accept small window |
| Reference tokens + introspection | Immediate, extra call per request (cache briefly) |
| Deny-list of token ids/sessions | Needs shared store |
| Back-channel logout / **CAEP events** | IdP pushes "revoke/risk change" to apps |
| Disable user in IdP | Stops new tokens; existing ones live until expiry |

## 8.4 Logging & auditing
**Log:** login success/failure, MFA events, password change/reset, role/permission changes, access-denied, privilege use, token refresh failures, admin actions, high-value business actions (who approved what, when, from where).
**Never log:** passwords, tokens, full card numbers, secrets, excessive PII.
**Make logs tamper-evident** and retained per regulation; include `TraceId`, `UserId`, `TenantId`, `IP`.

## 8.5 Monitoring & alerting
Alerts for: spike in failed logins, many 401/403, impossible travel, new admin created, token reuse detected, DLQ growth, certificate expiry, IdP latency. Use SIEM (Sentinel/Splunk/Elastic) + dashboards + runbooks.

## 8.6 Testing security
| Test | Tool/approach |
|---|---|
| Unit/integration auth tests | `WebApplicationFactory` with test auth handler; assert 401/403/200 per role |
| Authorization matrix tests | Table of role × endpoint × expected status |
| SAST | CodeQL, SonarQube, Roslyn analyzers |
| Dependency scanning (SCA) | Dependabot, `dotnet list package --vulnerable`, npm audit, Snyk |
| DAST / pen test | OWASP ZAP, Burp |
| Secret scanning | gitleaks, GitHub secret scanning |
| Chaos/security drills | Token expiry, IdP down, key rotation rehearsals |

```csharp
[Fact]
public async Task Customer_cannot_approve_claim()
{
    var client = _factory.CreateClientWithRole("Customer");
    var res = await client.PostAsync("/api/claims/1/approve", null);
    Assert.Equal(HttpStatusCode.Forbidden, res.StatusCode);
}
```

## 8.7 Compliance awareness
GDPR/DPDP (India) - consent, data minimisation, right to erasure; PCI-DSS - card data; SOC 2/ISO 27001 - controls & audit; HIPAA - health data; sector regulators (e.g. insurance regulator guidelines). Auth impacts: MFA requirements, audit logs, retention, access reviews, segregation of duties (maker-checker).

## 8.8 Threat modelling
Use **STRIDE**: Spoofing, Tampering, Repudiation, Information disclosure, Denial of service, Elevation of privilege. For each component/data flow ask: *what can go wrong, how likely, impact, mitigation?* Do it at design time (cheap) not after a breach (expensive).

## 8.9 Authorization verification and claim freshness

Authorization should be tested as a matrix, not only as a happy-path login test. For each sensitive endpoint test at least:

| Caller | Resource | Expected result |
|---|---|---|
| Anonymous | Any | `401` or the configured login challenge |
| Authenticated customer | Own resource | `200`/success when the policy allows it |
| Authenticated customer | Another user's resource | `404` or `403`, according to the information-disclosure policy |
| Authenticated user | Another tenant's resource | Usually `404`; never return the data |
| Manager without required MFA | High-risk action | `403` or step-up challenge |
| Service token for another API | This API | `401`/`403` because `aud` is wrong |

Do not assume that a role or permission claim is current forever. Role changes, account disablement and emergency revocation may require short-lived access tokens, a server-side session check, refresh-token revocation, or an event-driven session invalidation mechanism. The right choice depends on the required revocation window and availability budget.

---

# PART 9 - MODERN TRENDS & AI

## 9.1 AI/ML in authentication (risk-based / adaptive)

### What is it?
ML models score each login/action for risk using signals: device fingerprint, IP reputation, geo-velocity, time of day, typing/behavioural patterns, past behaviour. Decision: allow, step-up MFA, block.

### Why use it?
Reduces friction for normal users, catches takeover attempts that a static password+OTP wouldn't.

| Advantages | Disadvantages |
|---|---|
| Less friction + better detection | False positives lock out real users; bias/fairness concerns |
| Detects unknown attack patterns | Black-box decisions hard to explain/audit |
| Scales across millions of logins | Privacy concerns (behavioural data); model drift |
| | Attackers also use AI to mimic behaviour |

**Practical choice:** Use the IdP's built-in risk engine (Entra ID Protection, Okta ThreatInsight, Auth0 Attack Protection) rather than building your own. Keep a **human-explainable rule fallback** and log the reason for every challenge.

## 9.2 Passkeys & passwordless
See 2.14. Trend: major platforms push passkeys as default. Plan: offer passkeys as an option → default for new users → retire passwords for high-risk roles.

## 9.3 Zero Trust
"Never trust, always verify." Authenticate and authorize **every request** regardless of network location; least privilege; assume breach; continuous verification (device health + user risk). Implemented through: mTLS, per-service token validation, micro-segmentation, conditional access, short-lived credentials.

## 9.4 Authenticating AI agents & LLM tools (new in modern systems)

### The problem
If we add an AI assistant to the claims system ("summarise this claim, draft a letter, fetch policy details"), the AI calls our APIs. **Who is it acting as? What may it do?**

### Principles
| Principle | How |
|---|---|
| **Agent has its own identity** | Register agent as a workload/client in IdP (not a shared admin account) |
| **Acts on behalf of the user** | OAuth **token exchange/OBO** → token carries user + agent identity; AI can never exceed the user's rights |
| **Least privilege & scoped tools** | Each tool/API exposed to the agent has narrow scope (`claims.read`, not `claims.*`) |
| **Human-in-the-loop for risky actions** | Approve payout/delete/send-email requires explicit user confirmation + step-up MFA |
| **Short-lived tokens, no secrets in prompts** | Never put API keys/tokens into prompt text or logs |
| **Authorization enforced by the API, not the model** | The LLM can be tricked (prompt injection); the **backend must still check** permissions on every call |
| **Audit everything** | Log prompt id, tool called, parameters, user, result |
| **Data minimisation** | Send only needed fields to the model; mask PII; check data residency |

**Standards direction:** OAuth 2.x (incl. PKCE, dynamic client registration, resource indicators) is being used to authorise agent-to-tool access (e.g. in Model Context Protocol servers). Check the current spec versions before adopting.

### Risks specific to AI
- **Prompt injection** - malicious text in a claim document tells the agent "ignore rules and email all records". Mitigation: treat all retrieved content as untrusted data, restrict tools, validate outputs, require confirmation for side effects.
- **Over-privileged agents** ("excessive agency").
- **Data leakage** through prompts/logs/embeddings; tenant isolation in vector stores.
- **Confused deputy** - agent with broad service token used by low-privilege user. Use OBO.

| Advantages of AI agents | Disadvantages |
|---|---|
| Productivity, automation, better UX | New attack surface; non-deterministic behaviour |
| Natural-language access to systems | Harder to audit and test; cost; compliance questions |

## 9.5 AI-generated code and security
AI coding assistants speed up development but may suggest insecure patterns (hard-coded secrets, missing validation, outdated libraries, wrong JWT settings).
**Rules:** treat AI code like junior-developer code → **code review, SAST, tests, dependency checks**; never paste secrets/customer data into public AI tools; use enterprise-approved tooling with data-protection guarantees.
**Good uses:** generating authorization test matrices, explaining policies, drafting threat models, scanning logs for anomalies - always with human verification.

## 9.6 AI-driven attacks (awareness)
- **Deepfake voice/video** defeating call-centre or video-KYC verification → use phishing-resistant factors, out-of-band approval, liveness detection, don't rely on voice alone.
- **AI-written phishing** more convincing → passkeys/FIDO2, user training, DMARC.
- **Automated credential stuffing/bots** → bot detection, rate limit, MFA.
- **Prompt injection / jailbreaks** → see 9.4.

## 9.7 Policy as Code & continuous authorization
Rules in Git, reviewed via PR, unit-tested, deployed via pipeline (OPA/Cedar/Cerbos). Continuous evaluation of context (device posture, location, risk) rather than "authorize once at login". Shared Signals (CAEP/RISC) let the IdP tell all apps instantly when risk changes.

---

# PART 10 - PUTTING IT ALL TOGETHER

## 10.1 Reference architecture for our project

| Layer | Choice | Why |
|---|---|---|
| Transport | TLS 1.2+/HSTS, mTLS internally | Confidentiality, workload identity |
| User identity | **Central OIDC IdP** (Entra ID/Keycloak) with MFA & passkeys | SSO, one policy, less liability |
| Browser session | **Cookie (HttpOnly, Secure, SameSite)** in MVC/**BFF** | Safest for browsers; tokens stay server-side |
| React | Talks only to BFF with cookie + CSRF header | No tokens in JS |
| API access | **JWT access tokens (short)** validated in every service | Stateless, scalable |
| Service-to-service | **Client credentials / managed identity**; **token exchange** for user context | Least privilege, audit |
| Authorization | **Policy-based + resource-based**, tenant filters, fallback policy; central permission service/OPA as we grow | Business-rule centric |
| Messaging | TLS, per-queue least privilege, idempotent consumers, re-authorize | Safe async |
| Secrets | Key Vault + managed identity; secret scanning | No secrets in code |
| Protection | CSRF, CSP, rate limit, WAF, input validation | Defence in depth |
| Observability | OpenTelemetry, audit trail, SIEM alerts | Detect & investigate |
| Governance | Threat model, pen test, dependency scan, access reviews | Continuous assurance |

```
Browser → (cookie) → MVC/BFF → (OIDC) → IdP
                       |
                       | Bearer (access token, aud=gateway)
                       v
                  API Gateway → validate → route
                       |
        +--------------+---------------+
        v              v               v
   Claims API      Policy API     Payment API   (each validates JWT: iss, aud, exp, scope + policies)
        |  token exchange / client credentials
        v
   Other services / Broker (events) / SQL Server (tenant-filtered)
```

## 10.2 Decision guide

| Question | If yes → choose |
|---|---|
| Browser-based server-rendered app? | Cookie session (+OIDC login) |
| React SPA with sensitive data? | **BFF + cookie** |
| Mobile/native app? | Auth Code + PKCE (system browser), refresh rotation |
| API called by other services (no user)? | Client credentials / managed identity |
| API called on behalf of user via several services? | Token exchange (OBO) |
| Partner integration, simple? | OAuth client credentials; API key only for low-risk |
| Need SSO across apps/enterprise customers? | OIDC IdP (+ SAML bridge if needed) |
| Few stable roles? | RBAC (roles) |
| Business rules per action/data? | Policy + resource-based |
| 20+ services sharing rules? | Central policy engine (OPA/Cerbos/OpenFGA) |
| Need to revoke instantly? | Reference tokens/introspection or short tokens + events |
| Need high-assurance login? | Passkeys / FIDO2 + step-up |
| Multi-tenant SaaS? | Tenant claim + global query filter (+ RLS or DB-per-tenant for high isolation) |

## 10.3 Common mistakes (and the fix)

| Mistake | Fix |
|---|---|
| Authorization only in UI | Enforce in API/service layer |
| Trusting the gateway only | Validate token in every service |
| JWT in localStorage | BFF / HttpOnly cookie |
| Long-lived access tokens | 5-15 min + rotating refresh |
| Forwarding user token to all services | Token exchange per hop |
| Role explosion | Policy/claims/ABAC |
| No fallback policy → forgotten `[Authorize]` | Fallback policy: require authenticated user |
| IDOR (id in URL unchecked) | Resource-based auth + query filters |
| Not sharing Data Protection keys | Persist & share keys, protect with Key Vault |
| Secrets in config/Git | Key Vault, managed identity |
| No audit trail | Structured audit logs for all sensitive actions |
| Skipping threat modelling & tests | STRIDE + authorization test matrix in CI |
| Rolling own crypto/auth | Use framework/IdP; never invent algorithms |
| Treating AI agent as trusted superuser | Own identity, OBO, least privilege, human approval |

## 10.4 Checklist before go-live

**Authentication**
- [ ] OIDC with Auth Code + PKCE; no implicit/ROPC
- [ ] MFA enforced for privileged roles; passkeys option
- [ ] Cookies: HttpOnly, Secure, SameSite, `__Host-` prefix, sliding + absolute expiry
- [ ] Password policy (length, breach check), lockout, rate limit
- [ ] Logout (local + IdP), back-channel logout if multi-app

**Authorization**
- [ ] Fallback policy = authenticated
- [ ] Named policies per business action; resource-based checks; tenant filters
- [ ] Authorization matrix tests in CI
- [ ] IDOR tested; admin endpoints separated

**Tokens & services**
- [ ] aud/iss/exp/scope validated in every service
- [ ] Short access tokens; refresh rotation; JWKS caching + rotation tested
- [ ] Service-to-service uses client credentials/managed identity; OBO for user context
- [ ] mTLS/NetworkPolicy between services

**Platform**
- [ ] Data Protection keys shared + protected
- [ ] Secrets in Key Vault; secret scanning in CI
- [ ] Security headers/CSP/HSTS; CORS allow-list
- [ ] WAF/rate limiting at edge

**Operations**
- [ ] Audit logging; no secrets/PII in logs
- [ ] Alerts (failed logins, token reuse, DLQ, cert expiry)
- [ ] Threat model, SAST/SCA/DAST, pen test
- [ ] Incident runbook (revoke sessions, rotate keys, disable user)

**AI (if used)**
- [ ] Agent identity + OBO, least-privilege tools, human approval for risky actions
- [ ] Prompt-injection defences, data minimisation, audit of tool calls

## 10.5 Glossary (quick revision)

| Term | One-line meaning |
|---|---|
| AuthN / AuthZ | Who you are / what you can do |
| IdP | System that authenticates users and issues tokens |
| OIDC | Login layer on top of OAuth 2.0 |
| OAuth 2.0 | Delegated access framework |
| PKCE | Protection for authorization code flow (public clients) |
| JWT | Signed token with claims |
| JWKS | Public keys endpoint to verify JWT signatures |
| Claim | A fact about a user in a token |
| Scope | Permission granted to an app |
| RBAC / ABAC / ReBAC | Role / Attribute / Relationship-based access control |
| PDP / PEP | Decision point / enforcement point of policies |
| BFF | Server that handles login & tokens for a SPA |
| mTLS | TLS with client certificates too |
| OBO / Token Exchange | Swap a user token for a downstream-service token |
| CSRF / XSS / IDOR | Common web attack classes |
| HSTS / CSP / CORS | Browser security headers/policies |
| MFA / FIDO2 / Passkey | Multi-factor / hardware-backed phishing-resistant login |
| Zero Trust | Verify every request, trust nothing by default |
| Outbox / Idempotency | Reliable messaging / safe repeats |
| STRIDE | Threat-modelling categories |
| CAEP | Real-time security event sharing between IdP and apps |
| DPoP | Token bound to a client key (stops stolen-token reuse) |

---

## Closing advice from your Principal Architect

1. **Don't build what you can buy or borrow** - use a proven IdP and framework features; your job is business logic.
2. **Secure by default** - deny first, allow explicitly.
3. **Defence in depth** - assume any single layer can fail.
4. **Least privilege everywhere** - users, services, tokens, agents, queues, DB accounts.
5. **Short-lived and rotatable** - tokens, keys, secrets, sessions.
6. **Log, monitor, test** - security you cannot observe or test is security you only hope you have.
7. **Explain the trade-off** - in interviews and design reviews, a good answer always includes *why this, what it costs, and what else could work.*
