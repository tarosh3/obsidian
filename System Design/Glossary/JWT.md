---
title: "JWT (JSON Web Token)"
aliases: [JWT, JSON Web Token]
tags: [system-design, glossary]
---

# JWT — JSON Web Token

> [!info] Quick definition
> A self-contained token — `header.payload.signature`, each base64-encoded — used for stateless authentication. The server can verify it (via the signature) without a database lookup, since all the claims (user ID, roles, expiry) are embedded in the payload.

**Pitfall to know cold:** the payload is **encoded, not encrypted** — anyone can base64-decode it and read the claims (never put secrets in a JWT payload). Another classic pitfall: the `alg: none` attack, where some libraries historically accepted a token claiming "no signature algorithm," letting an attacker forge tokens trivially if not properly guarded against.

> [!tip] Full chapter
> The revocation problem, refresh-token rotation with theft detection, and the full OAuth2 flow: [[CS Fundamentals/08 - Security/Authentication & Authorization|Authentication & Authorization]].
