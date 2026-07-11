---
title: "OAuth2"
aliases: [OAuth, OAuth2, OAuth 2.0]
tags: [system-design, glossary]
---

# OAuth2

> [!info] Quick definition
> A delegation protocol letting a user grant a third-party app **limited access** to their data on another service, without ever sharing their password with that third party. "Log in with Google" is OAuth2 in action.

The flow worth knowing cold: **Authorization Code Flow** — app redirects user to the provider (Google/GitHub) → user approves → provider redirects back with a short-lived **code** → app's backend exchanges that code (server-to-server, using a client secret) for an **access token** — the code-for-token exchange happens off the browser specifically so the token itself is never exposed in a redirect URL.

> [!tip] Full chapter
> PKCE (and the attack it prevents), Client Credentials flow, OIDC, and RBAC vs. ABAC authorization: [[CS Fundamentals/Security/Authentication & Authorization|Authentication & Authorization]].
