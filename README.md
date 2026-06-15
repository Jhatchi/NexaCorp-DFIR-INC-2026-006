# NexaCorp DFIR: INC-2026-006

Stored Cross-Site Scripting (XSS) leading to session hijacking on an internal web portal. Blue Team forensic investigation and Suricata detection engineering.

This repository documents a SOC analyst engagement carried out as part of the BeCode Cybersecurity Bootcamp (promotion 2025-2026). It reconstructs a full intrusion from network and log evidence, then delivers a validated set of network detection rules. It is the sixth incident in the NexaCorp DFIR series.

> Training engagement on a controlled lab. No production system, customer data, or third party was involved. All IP addresses, hostnames, and accounts belong to the BeCode SOC training environment.

---

## Incident at a glance

| Field | Value |
|---|---|
| Reference | INC-2026-006 |
| Affected host | bru-web-02 (NexaPortal, 192.168.10.24) |
| Attacker | 91.92.100.45 (external, Firefox on Linux) |
| Root cause | Stored XSS (CWE-79) on an unsanitised feedback feature |
| Outcome | Session cookie theft, session hijack of p.dumont, unauthorized account m.renard |
| Date of incident | 18 June 2026 |
| Evidence | web_access.log, attack.pcap |
| Phases | Phase 1 forensic analysis, Phase 2 Suricata detection |
| Related incidents | INC-2026-004, INC-2026-005 (web application incidents) |

---

## Kill chain summary

The attacker compromised an employee session without ever needing a password:

1. **Reconnaissance** of the portal from 91.92.100.45 (first contact 08:32 UTC).
2. **Injection**: a feedback entry submitted to `POST /portal/feedback.php` carrying a `<script>` tag instead of text.
3. **Storage**: NexaPortal saved the payload in its database and later rendered it back without output encoding (the defining trait of stored XSS).
4. **Trigger**: employee p.dumont loaded the feedback page (15:50:58 UTC); the stored script executed in the context of the authenticated session.
5. **Exfiltration**: the script read `document.cookie`, Base64-encoded it with `btoa`, and beaconed it to `91.92.100.45:8080/collect` (15:51:00 UTC).
6. **Hijack**: the attacker replayed the stolen `PHPSESSID` and accessed the portal as p.dumont (approx 15:52 UTC).

Investigation also surfaced an account, `m.renard`, that authenticated from the attacker IP and matches no NexaCorp employee directory entry. Its origin is not fully established (attacker-created foothold, or compromised pre-existing account); in both cases it must be treated as unauthorized and removed. This nuance is stated as fact, not forced into a single conclusion, in the report.

---

## Key metrics

| Metric | Value |
|---|---|
| Attacker IP | 91.92.100.45 (external) |
| Injection vector | POST /portal/feedback.php |
| Victim account | p.dumont |
| Stolen session cookie | PHPSESSID (Base64-exfiltrated via beacon) |
| Unauthorized account | m.renard (origin undetermined) |
| Suricata detection rules authored | 2 (SID 1000001 to 1000002) |
| Rules firing against the capture | 2 of 2 (1 injection, 1 beacon) |

---

## Repository structure

```
NexaCorp-DFIR-INC-2026-006/
├── README.md                      This file
├── LICENSE
├── .gitignore
├── .markdownlint.json
├── .github/
│   └── workflows/
│       └── ci.yml                 markdownlint + typography validation
├── reports/
│   └── INC-2026-006_Findings_Report.md    Markdown source of the report, readable on GitHub
├── detection/
│   ├── lab.rules                  The 2 Suricata rules (SID 1000001-1000002)
│   └── README.md                  Per-rule logic and validation
├── evidence-summary/
│   └── ioc-summary.md             Indicators of compromise, SIEM-ingestible format
├── methodology/
│   ├── attack-timeline.md         Full request-level timeline
│   └── attck-mapping.md           MITRE ATT&CK mapping table with section references
└── notes/
    └── journal.md                 Investigation journal (post-analysis reconstruction)
```

The findings report PDF (`reports/INC-2026-006_Findings_Report.pdf`) is added on publication.

---

## Detection engineering highlight

Two rules, one per critical stage of the chain, each targeting the Suricata buffer suited to its intent:

| SID | Detects | Buffer | Match |
|---|---|---|---|
| 1000001 | XSS injection in feedback submission | http.request_body | `script` in POST body |
| 1000002 | Cookie exfiltration beacon | http.uri | `/collect` to 91.92.100.45:8080 |

Both rules fire against the incident capture for a total of 2 alerts (one injection, one beacon), covering the full attack chain with no false positives on the captured traffic. Full detail in `detection/README.md`.

---

## NexaCorp DFIR series

- [INC-2026-001](https://github.com/Jhatchi/NexaCorp-DFIR-INC-2026-001): Linux infrastructure compromise (vsftpd backdoor, Caldera C2)
- [INC-2026-002](https://github.com/Jhatchi/NexaCorp-DFIR-INC-2026-002): privilege escalation and persistence (Tor SSH, SUID, backdoor account)
- [INC-2026-003](https://github.com/Jhatchi/NexaCorp-DFIR-INC-2026-003): month-1 cross-incident assessment
- [INC-2026-004](https://github.com/Jhatchi/NexaCorp-DFIR-INC-2026-004): SQL injection (web portal)
- [INC-2026-005](https://github.com/Jhatchi/NexaCorp-DFIR-INC-2026-005): OS command injection and web shell (web portal)
- **INC-2026-006**: this repository

---

## Standards and frameworks

NIST SP 800-61r2 (incident handling), NIST SP 800-86 (forensic techniques), SANS PICERL, MITRE ATT&CK Enterprise v15, CWE-79 (Cross-Site Scripting), OWASP Top 10 A03:2021 (injection).

---

*Author: Johan-Emmanuel Hatchi · BeCode Cybersecurity Bootcamp 2025-2026 · [linkedin.com/in/johan-emmanuel-hatchi](https://linkedin.com/in/johan-emmanuel-hatchi)*
