# Tenda F6 (N300) — Administrator Credential Exposure via Cleartext HTTP and Reversible Cookie Storage

| | |
|---|---|
| **Vendor** | Tenda |
| **Product** | F6 (N300) Wireless Router |
| **Affected version** | Firmware V12.02.01.73 (Hardware V5.0) |
| **Latest version at disclosure** | V12.02.02.77 (not tested) |
| **Vulnerability type** | Cleartext Transmission of Sensitive Information (CWE-319); also CWE-312 / CWE-522 |
| **Severity** | High — CVSS 3.1 Base 8.3 |
| **CVSS vector** | `AV:A/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H` |
| **Status** | Reported to vendor 2026-10-09 — awaiting response |

## Summary

The web management interface of the Tenda F6 (N300) exposes administrator
credentials through two compounding weaknesses:

1. **Cleartext transmission.** The interface is served only over HTTP, with no
   TLS/HTTPS option. The login request and the session cookie are transmitted
   unencrypted and can be captured by any attacker able to observe local network
   traffic (e.g. shared Wi-Fi, ARP spoofing).
2. **Reversible credential storage in the session cookie.** After login, the
   session cookie `ecos_pw` contains the administrator password encoded with
   Base64 — an encoding, not encryption — which is trivially reversible.

Together these allow a network-adjacent attacker to recover the administrator
password and authenticate, obtaining full administrative control of the device.

## Affected component

- Web management interface (default `http://192.168.0.1`), served over HTTP only
- Session cookie: `ecos_pw`

## Proof of Concept

A captured request to the management interface contains the session cookie in
cleartext:

```http
POST /goform/sysRestore HTTP/1.1
Host: 192.168.0.1
Content-Type: application/x-www-form-urlencoded
Cookie: bLanguage=en; ecos_pw=YWRtaW4=
Content-Length: 33

module1=sysOperate&action=restore
```

The cookie value is Base64 and decodes directly to the administrator password:

```
$ echo 'YWRtaW4=' | base64 -d
admin
```

*(In this test the password was set to `admin`; the value demonstrates that the
plaintext password is recoverable from the cookie by simple Base64 decoding.)*

Because the interface runs over HTTP, this cookie is also transmitted in
cleartext on every request and can be captured by passive network sniffing.

Screenshot evidence: [`ecos_pw-cookie.png`](./ecos_pw-cookie.png)

## Steps to reproduce

1. Intercept traffic to `http://192.168.0.1` using an HTTP proxy (e.g. Burp
   Suite) or a network analyzer (e.g. Wireshark) while an administrator
   authenticates or uses the interface.
2. Extract the `ecos_pw` cookie value from the captured HTTP request.
3. Base64-decode the value to recover the administrator password.
4. Authenticate to the management interface with the recovered password to
   confirm full administrative access.

## Impact

- **Administrator credential disclosure** — the password is recoverable from
  both the cleartext HTTP traffic and the reversible `ecos_pw` cookie.
- **Full administrative compromise** — confirmed by authenticating with the
  recovered credential, granting complete control of the router (settings,
  reboot, factory reset, Wi-Fi configuration).
- **No user interaction required** — the attacker only needs to observe a login
  or any authenticated request on the network.

## Remediation

- Serve the management interface exclusively over HTTPS/TLS; redirect HTTP to
  HTTPS and enable HSTS.
- Never store the password in a cookie. Use a random, opaque, server-side
  session identifier.
- Store passwords only as salted hashes (bcrypt/Argon2); never in a reversible
  form anywhere on the client.
- Set session cookies to `Secure`, `HttpOnly`, and `SameSite`.

## Disclosure timeline

| Date | Event |
|---|---|
| 2026-10-09 | Vulnerability discovered |
| 2026-10-09 | Reported to Tenda (productsecure@tenda.cn) |
| *pending* | Advisory published |

## Credits

Discovered by [0xWKZER](https://github.com/0xWKZER) and [Friend's Name].

## Disclaimer

This advisory is published for educational and defensive purposes. Testing was
performed only on devices owned by the researchers. Do not use this information
against systems you do not own or are not authorized to test.
