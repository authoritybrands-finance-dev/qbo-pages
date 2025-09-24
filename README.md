# QuickBooks OAuth Integration – Architecture & Rationale

## A. The Initial Issue

When integrating with QuickBooks Online, Intuit enforces strict production requirements for OAuth 2.0 applications:

- The Redirect URI must be HTTPS and must resolve publicly on the internet.
- `localhost` URIs are explicitly disallowed in production apps.

Self-hosted servers (e.g., Hetzner bare metal) were considered non-viable for this flow because:

- They require ongoing TLS certificate management.
- They introduce security and uptime risks (any downtime blocks OAuth flows).
- They can’t easily provide globally reachable URLs acceptable to Intuit reviewers.

At the same time, we had specific goals:

- Keep it free or extremely low-cost.
- Zero infrastructure maintenance (no patching, no custom NGINX, no monitoring overhead).
- Strict security requirements: client secrets must never be exposed in front-end code or GitHub.
- Low friction for users (franchisees or internal staff should just click “Connect” and be done).

This created a classic problem: how to provide a public, production-valid, secure redirect environment for Intuit OAuth flows without taking on hosting burdens or leaking secrets.

---

## B. The Solution and Why

Our architecture combined **GitHub Pages** (for the public static front-end) and **Cloudflare Workers** (for secure backend token exchange). This design checked all boxes: free, low-maintenance, production-compliant, and secure.

### 1. GitHub Pages – Static Public Redirect Pages

We hosted three simple static HTML files via GitHub Pages:

- `qbo-launch.html` – A “connected” landing page (human-friendly).
- `qbo-disconnect.html` – A “disconnected” landing page (used when revoking).
- `redirect.html` – The registered OAuth Redirect URI that Intuit calls after user consent.

**Why GitHub Pages?**

✅ Meets Intuit’s production HTTPS requirement (`https://<org>.github.io/...` is globally valid).  
✅ Free hosting with automatic SSL — no cost, no certificate renewal.  
✅ Zero maintenance — once deployed, there’s no server to monitor.  
✅ Separation of concerns — only handles safe, static tasks (confirmation messages and forwarding).

**Security hardening on `redirect.html`:**

- **Content Security Policy (CSP):** Only allow outbound requests to our Cloudflare Worker. Blocks data exfiltration to any other host.
- **Referrer Policy:** Ensure that authorization codes cannot leak via HTTP headers.
- **Query Scrubbing:** Strip `code` and `realmId` from the browser bar using `history.replaceState`. This prevents accidental leaks in screenshots, bookmarks, or analytics logs.
- **Minimal UX:** Only displays a success checkmark. Tokens are never shown to the user.

---

### 2. Cloudflare Workers – Secure Backend Exchange

Intuit requires the authorization code to be exchanged for tokens server-side. That means you must send `client_id` + `client_secret` securely — impossible from JavaScript running in a browser.

We used Cloudflare Workers as the backend to perform this exchange:

**Public URL:**  
`https://<service>.<account>.workers.dev/api/qbo/oauth/callback`  
(registered with Intuit as the Redirect URI).

**Environment secrets:**  
Client IDs and secrets are stored in Cloudflare’s secret store. They never appear in GitHub or front-end code.

**CORS lock-down:**  
The Worker only accepts requests from `https://<org>.github.io`. Any other origin is rejected.

**Responsibilities:**

- Accept `POST` from `redirect.html`.
- Exchange `code` + `client_secret` with Intuit’s `/token` endpoint.
- Return access + refresh tokens (or errors) securely.
- Optionally persist tokens in Workers KV or forward to downstream DBs.

**Why Cloudflare Workers?**

✅ Global edge deployment → consistent low-latency worldwide.  
✅ No server management → auto-scales and runs free (under generous limits).  
✅ Permanent HTTPS URL (`workers.dev`) → Intuit accepts it in production.  
✅ Secret isolation → Client secrets live in Cloudflare, never exposed in GitHub or browsers.  
✅ Security → tight CORS, no inbound ports, minimal attack surface.

---

### 3. Optional Enhancements (Future-Proofing)

The architecture was designed to optionally support:

- **PKCE (Proof Key for Code Exchange):** Extra security for “public” clients. Prevents intercepted codes from being reused.
- **Signed state values (HMAC/JWT):** Ensures the OAuth flow can’t be hijacked or replayed. Links every redirect to its initiating request.
- **Token persistence** in Workers KV, D1, or external databases. Currently tokens are stored downstream, but this gives flexibility for scaling.

---

### 4. Why This Matters

This setup solves the exact pain points we started with:

- Meets Intuit’s production rules (HTTPS, public URL, no localhost).
- Zero hosting burden (GitHub Pages + Workers are free, global, and auto-maintained).
- Security guarantees (client secrets never leave the server side, authorization codes never linger in the browser).
- Future-proof (PKCE, persistent token storage, logging can be added without redesign).

**In short**: we achieved secure, compliant OAuth integration without spinning up or maintaining a single custom server.

---

### 5. Final Flow

1. User clicks “Connect to QuickBooks”.
2. Intuit redirects back to GitHub Pages `redirect.html` with `code`, `state`, and `realmId`.
3. Redirect page scrubs the URL, then `POST`s the data to Cloudflare Worker.
4. Worker performs secure `code → token` exchange using stored client secrets.
5. Worker responds with tokens (and optionally persists them).
6. Redirect page simply shows:  
   **✅ Connected. You can close this tab.**

---

## TL;DR for Future Developers
- **Problem:** Intuit requires production OAuth Redirect URIs to be HTTPS/public; we couldn’t use localhost or raw Hetzner servers.
- **Solution:** Static GitHub Pages for the user-facing flow + Cloudflare Workers for secure token exchange.
- **Why:** Free, zero-maintenance, Intuit-compliant, globally available, and secure against data leaks.

