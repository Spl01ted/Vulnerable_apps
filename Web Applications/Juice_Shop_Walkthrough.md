# OWASP Juice Shop — Vulnerability Walkthrough

> **Disclaimer:** This walkthrough is intended for educational purposes only. All findings were discovered against a local, intentionally vulnerable application. Never test techniques against systems you do not own or have explicit permission to test.

---

## Table of Contents

1. [DOM XSS — Search Field](#1-dom-xss--search-field)
2. [SQL Injection — Authentication Bypass](#2-sql-injection--authentication-bypass)
3. [JWT None Algorithm Attack](#3-jwt-none-algorithm-attack)
4. [BOLA — Basket Functionality](#4-bola--basket-functionality)
5. [BOLA — Order Confirmation](#5-bola--order-confirmation)
6. [FTP Directory Exposure](#6-ftp-directory-exposure)

---

## 1. DOM XSS — Search Field

| Field | Detail |
|---|---|
| **Vulnerability** | DOM-Based Cross-Site Scripting (XSS) |
| **Location** | Search field on the root page |
| **Severity** | High |

### Summary

The application's search field reflects user input directly into the DOM without sanitization. By injecting an HTML payload, an attacker can execute arbitrary JavaScript in the victim's browser — enabling session hijacking, credential theft, or malicious redirects.

### Steps to Reproduce

1. Navigate to the root page of the application.

2. Paste the following payload into the search field:

```html
<img src=x onerror=alert(1)>
```

3. The browser renders the injected element, the `src` attribute fails to load, and the `onerror` handler fires — executing `alert(1)`.

![DOM XSS triggered via search field](https://github.com/user-attachments/assets/6a25ba39-67c8-4ff3-8c95-89d4fd5e56a8)

### Mitigation

- Sanitize all user-supplied input before inserting it into the DOM. Use a library such as [DOMPurify](https://github.com/cure53/DOMPurify) for client-side sanitization.
- Avoid using dangerous DOM sinks such as `innerHTML`, `document.write()`, or `eval()` with unsanitized data. Prefer safe alternatives like `textContent`.
- Implement a strict **Content Security Policy (CSP)** header to block inline script execution as a defense-in-depth measure.

---

## 2. SQL Injection — Authentication Bypass

| Field | Detail |
|---|---|
| **Vulnerability** | SQL Injection — Authentication Bypass & Privilege Escalation |
| **Endpoint** | Login form |
| **Severity** | Critical |

### Summary

The login form passes user-supplied input directly into a SQL query without parameterization. An attacker can inject SQL syntax to comment out the password check, bypassing authentication for any known account. The same technique can be extended to gain access to the first account in the database — typically the admin.

### Steps to Reproduce

#### Bypass authentication for a known account

1. Create an account and navigate to the login page.

2. In the **email** field, enter your valid email address followed by the injection payload:

```
your@email.com' --
```

3. Enter any value in the **password** field (it will be ignored).

4. Click **Login**. The SQL comment (`--`) causes the database to discard the password check entirely, and you are logged in.

![SQL injection authentication bypass](https://github.com/user-attachments/assets/8793038f-fb29-438a-9635-9d8eee7457bb)

#### Escalate to admin access

Leave the email field empty and use the following payload instead:

```
' OR 1=1--
```

This causes the query to match the **first row** in the users table — which is typically the admin account — granting full administrative access without any credentials.

### Mitigation

- Use **parameterized queries** (prepared statements) for all database interactions. Never concatenate user input into SQL strings.
- Apply the **principle of least privilege** to database accounts used by the application.
- Implement login rate limiting and account lockout to reduce the impact of automated injection attempts.

```javascript
// FIXED — parameterized query
db.query('SELECT * FROM users WHERE email = ? AND password = ?', [email, hashedPassword]);
```

---

## 3. JWT None Algorithm Attack

| Field | Detail |
|---|---|
| **Vulnerability** | JWT Algorithm Confusion (None Algorithm) |
| **Endpoint** | `GET /profile` (and any JWT-authenticated endpoint) |
| **Severity** | Critical |

### Summary

The application accepts JWT tokens where the `alg` header field is set to `none`, meaning no signature is required. An attacker can forge a token for any user by simply modifying the payload and removing the signature — without needing the signing secret.

### Steps to Reproduce

1. Create **two** accounts (Account A and Account B) and log in to Account A.

2. Navigate to your profile page (`/profile`) and capture the outgoing request. Copy the JWT from the `Authorization` header.

3. Base64-decode the JWT header and payload. The decoded payload will look similar to:

```json
{
  "id": 22,
  "email": "account-a@example.com",
  "iat": 1776237401
}
```

![JWT decoded - id field visible](https://github.com/user-attachments/assets/565129a4-6f23-49cd-aafc-bb8a079c4b51)

4. Log out and log in to **Account B**. Repeat the steps above and note Account B's `id` value (e.g., `23`).

5. Log back in to **Account A** and craft a new JWT:
   - Change the `alg` header field to `none`
   - Change the `id` field in the payload to Account B's ID (e.g., `23`)
   - Remove the signature entirely (keep the trailing `.` after the payload)

The resulting token structure: `<base64-header>.<base64-payload>.`

6. Send a request to `/profile` with the forged token. The server accepts it and returns Account B's profile data.

### Mitigation

- Explicitly reject tokens with `"alg": "none"` on the server. Most modern JWT libraries offer a configuration option to specify an allowlist of accepted algorithms.
- Never allow the client to dictate which algorithm is used for verification.

```javascript
// FIXED — explicitly allowlist the expected algorithm
jwt.verify(token, secret, { algorithms: ['HS256'] });
```

---

## 4. BOLA — Basket Functionality

| Field | Detail |
|---|---|
| **Vulnerability** | Broken Object Level Authorization (BOLA / IDOR) |
| **Endpoint** | `GET /rest/basket/{id}` |
| **Severity** | High |

### Summary

The basket endpoint accepts a numeric `id` path parameter to identify which basket to return. The server does not verify that the requesting user owns the basket, allowing any authenticated user to view another user's basket contents by changing the ID.

### Steps to Reproduce

1. Log in and navigate to your **Basket**. Intercept the request:

```http
GET /rest/basket/6 HTTP/1.1
Host: localhost:3000
Authorization: Bearer <your-token-here>
```

2. Change the `id` in the URL to a different integer (e.g., `5`, `4`, `3`).

3. The server returns the basket belonging to that other user.

**Your basket before the attack:**

![Own basket contents](https://github.com/user-attachments/assets/29e9e554-62b1-4f87-8ca1-ca97abf44b23)

**Another user's basket after changing the ID:**

![Other user's basket exposed](https://github.com/user-attachments/assets/8da52564-fffb-4884-befe-633c2470bd52)

### Mitigation

On the server side, verify that the basket's `userId` matches the authenticated user's ID before returning any data. Return `403 Forbidden` if they do not match:

```javascript
// FIXED — verify ownership before returning basket
router.get('/rest/basket/:id', authenticate, (req, res) => {
    const basket = db.baskets.findById(req.params.id);
    if (!basket || basket.userId !== req.user.id) {
        return res.status(403).json({ error: 'Forbidden' });
    }
    res.json(basket);
});
```

---

## 5. BOLA — Order Confirmation

| Field | Detail |
|---|---|
| **Vulnerability** | Broken Object Level Authorization (BOLA / IDOR) |
| **Endpoint** | `POST /rest/basket/{id}/checkout` |
| **Severity** | High |

### Summary

The checkout endpoint is similarly missing authorization checks. By changing the basket `id` in the URL, an authenticated user can trigger and view the order confirmation for another user's basket — exposing their order details and potentially their payment information.

### Steps to Reproduce

1. Add a product to your basket, add a payment card, and proceed to checkout. Intercept the checkout request:

```http
POST /rest/basket/6/checkout HTTP/1.1
Host: localhost:3000
Authorization: Bearer <your-token-here>
```

2. Change the `{id}` in the URL to another user's basket ID.

3. The server processes the request and returns the other user's order confirmation.

**Order confirmation for your own basket:**

![Own order confirmation](https://github.com/user-attachments/assets/1bd09059-1bcb-4eeb-a47a-be8c3b4a7c22)

**Another user's order confirmation after changing the ID:**

![Other user's order confirmation exposed](https://github.com/user-attachments/assets/853e973d-5e4a-4d76-9a1e-898364711e93)

### Mitigation

Apply the same ownership check as in Finding #4. Both the basket view and checkout endpoints must verify that the requested basket belongs to the authenticated user before processing:

```javascript
// FIXED — verify ownership before processing checkout
router.post('/rest/basket/:id/checkout', authenticate, (req, res) => {
    const basket = db.baskets.findById(req.params.id);
    if (!basket || basket.userId !== req.user.id) {
        return res.status(403).json({ error: 'Forbidden' });
    }
    // proceed with checkout
});
```

---

## 6. FTP Directory Exposure

| Field | Detail |
|---|---|
| **Vulnerability** | Security Misconfiguration — Exposed FTP Directory |
| **Endpoint** | `GET /ftp` |
| **Severity** | Medium |

### Summary

The application exposes an FTP directory listing at `/ftp` with no authentication required. This publicly accessible directory may contain sensitive files such as backup files, configuration data, internal documents, or source code artifacts.

### Steps to Reproduce

1. Navigate to the following URL in your browser (no authentication required):

```
http://localhost:3000/ftp
```

The server returns a full directory listing, allowing any visitor to browse and download the files hosted there.

### Mitigation

- Remove the `/ftp` route from the public-facing application entirely, or place it behind authentication and authorization checks.
- Audit all files currently exposed in the directory and rotate any credentials or secrets found within them.
- Disable directory listing at the web server level (e.g., `Options -Indexes` in Apache, `autoindex off` in Nginx) as a defense-in-depth measure.
- Store sensitive files outside the web root so they cannot be served directly by the HTTP server under any circumstances.

---

## Summary Table

| # | Vulnerability | Location / Endpoint | Severity |
|---|---|---|---|
| 1 | DOM-Based XSS | Search field | High |
| 2 | SQL Injection — Auth Bypass & Privilege Escalation | Login form | Critical |
| 3 | JWT None Algorithm Attack | All authenticated endpoints | Critical |
| 4 | BOLA — Basket View | `GET /rest/basket/{id}` | High |
| 5 | BOLA — Order Checkout | `POST /rest/basket/{id}/checkout` | High |
| 6 | FTP Directory Exposure | `GET /ftp` | Medium |

---

*Generated against OWASP Juice Shop — a deliberately insecure web application for security training and awareness.*
