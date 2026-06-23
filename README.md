# NexaCorp DFIR: INC-2026-006 - Stored XSS and Session Hijacking

Forensic investigation of a stored Cross-Site Scripting (XSS) attack against NexaPortal, NexaCorp's internal employee web portal (`bru-web-02`). An external attacker planted a malicious `<script>` in the portal's feedback feature; because the input was stored and rendered without output encoding, the script executed in an employee's authenticated session, read the session cookie, and beaconed it (Base64-encoded) to an attacker-controlled collector. The attacker then replayed the stolen session to access the portal as that employee, with no password required. Conducted as a solo engagement during the BeCode Brussels Blue & Red Team bootcamp (Mission 06), as the continuation of [INC-2026-001](https://github.com/Jhatchi/NexaCorp-DFIR-INC-2026-001), [INC-2026-002](https://github.com/Jhatchi/NexaCorp-DFIR-INC-2026-002), [INC-2026-003](https://github.com/Jhatchi/NexaCorp-DFIR-INC-2026-003), [INC-2026-004](https://github.com/Jhatchi/NexaCorp-DFIR-INC-2026-004), and [INC-2026-005](https://github.com/Jhatchi/NexaCorp-DFIR-INC-2026-005).

[![ci](https://github.com/Jhatchi/NexaCorp-DFIR-INC-2026-006/actions/workflows/ci.yml/badge.svg)](https://github.com/Jhatchi/NexaCorp-DFIR-INC-2026-006/actions/workflows/ci.yml)
[![Methodology](https://img.shields.io/badge/methodology-NIST%20SP%20800--61r2-blue.svg)](#methodology)
[![Framework](https://img.shields.io/badge/framework-MITRE%20ATT%26CK-red.svg)](https://attack.mitre.org/)
[![Detection](https://img.shields.io/badge/Suricata-2%20rules%20validated-green.svg)](detection/local-xss.rules)
[![CWE](https://img.shields.io/badge/CWE--79-Stored%20XSS-orange.svg)](https://cwe.mitre.org/data/definitions/79.html)
[![License](https://img.shields.io/badge/license-MIT-yellow.svg)](LICENSE)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Johan--Emmanuel%20Hatchi-0A66C2?logo=linkedin&logoColor=white)](https://www.linkedin.com/in/johan-emmanuel-hatchi/)

This repository documents a SOC analyst engagement carried out as part of the BeCode Cybersecurity Bootcamp (promotion 2025-2026). It reconstructs a full intrusion from network and log evidence, then delivers a validated set of network detection rules. It is the sixth incident in the NexaCorp DFIR series.

## Contents

- [Operational notice](#operational-notice)
- [At a glance](#at-a-glance)
- [Engagement context](#engagement-context)
- [Executive summary](#executive-summary)
- [Kill chain summary](#kill-chain-summary)
- [How to read this repository](#how-to-read-this-repository)
- [Methodology](#methodology)
- [Tools used](#tools-used)
- [Detection engineering](#detection-engineering)
- [Repository layout](#repository-layout)
- [Reproducibility](#reproducibility)
- [Known limits](#known-limits)
- [NexaCorp DFIR series](#nexacorp-dfir-series)
- [Acknowledgments](#acknowledgments)
- [About](#about)
- [License](#license)

---

## Operational notice

This is a training engagement against fictitious infrastructure. NexaCorp Industries is a fictional client used as the scenario for the BeCode Brussels bootcamp. The host `bru-web-02` (NexaPortal) is an isolated lab environment. No production system, customer data, or third party was involved.

All IP addresses, hostnames, and accounts referenced here (`91.92.100.45`, `bru-web-02`, `p.dumont`, `m.renard`, and similar) are lab-local artifacts, not real-world threat intelligence. Do not feed them to a production SIEM as IOCs.

---

## At a glance

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

| Investigation output | Value |
|---|---|
| Attacker IP | 91.92.100.45 (external) |
| Injection vector | POST /portal/feedback.php |
| Victim account | p.dumont |
| Stolen session cookie | PHPSESSID (Base64-exfiltrated via beacon) |
| Unauthorized account | m.renard (origin undetermined) |
| Suricata detection rules authored | 2 (SID 1000001 to 1000002) |
| Rules firing against the capture | 2 of 2 (1 injection, 1 beacon) |

---

## Engagement context

**Scenario (fictional).** NexaCorp Industries reported a stored XSS incident on its internal web portal NexaPortal (`bru-web-02`, 192.168.10.24). The engagement was reported by Marc Wauters (IT Infrastructure Manager); the incident occurred on Wednesday 18 June 2026. The portal had been launched two weeks earlier without a security review.

**Scope.** Phase 1 is a forensic analysis of the evidence bundle (web_access.log, attack.pcap). Phase 2 is a Suricata detection-engineering phase: two rules authored and validated against the capture.

**Educational context.** Delivered during the BeCode Brussels Blue & Red Team bootcamp (November 2025 to September 2026) as Mission 06.

---

## Executive summary

On 18 June 2026, an external attacker (91.92.100.45) compromised an employee session on NexaCorp's internal portal NexaPortal. The attacker submitted a malicious feedback entry containing hidden JavaScript. Because the portal stored and displayed that feedback without sanitizing it, the script ran automatically in the browser of an employee (p.dumont) who later viewed the page, silently copied that employee's session identifier, and sent it to an attacker-controlled server.

Using the stolen session, the attacker accessed the portal as the legitimate employee without needing a password. Investigation also revealed an unauthorized account (m.renard) that matches no NexaCorp employee. The root cause is the absence of input sanitization and output encoding on the feedback feature. Two Suricata rules were authored to detect the injection and the exfiltration beacon.

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

## How to read this repository

| If you are a... | Start here | Time |
|---|---|---|
| **Recruiter or hiring manager** | This README + the [report](reports/INC-2026-006_Findings_Report.md) executive summary | 5 min |
| **SOC analyst evaluating fit** | [Report](reports/INC-2026-006_Findings_Report.md) technical analysis + [`evidence-summary/ioc-summary.md`](evidence-summary/ioc-summary.md) | 20 min |
| **Detection engineer** | [`detection/local-xss.rules`](detection/local-xss.rules) + [`detection/README.md`](detection/README.md) for per-rule logic and validation | 20 min |
| **DFIR practitioner** | Full [report](reports/INC-2026-006_Findings_Report.md) + [`methodology/attack-timeline.md`](methodology/attack-timeline.md) + [`notes/journal.md`](notes/journal.md) | 40 min |
| **Anyone who wants to grep, cite, or diff** | [Markdown source of the report](reports/INC-2026-006_Findings_Report.md) | as needed |

---

## Methodology

The engagement follows standard incident-response and detection frameworks:

- **NIST SP 800-61r2** (incident handling) and **NIST SP 800-86** (forensic techniques): structure the Detection & Analysis work.
- **SANS PICERL**: the Identification stage is the core of Phase 1 (log and PCAP correlation).
- **MITRE ATT&CK Enterprise v15**: every finding is mapped to one or more techniques (see [`methodology/attck-mapping.md`](methodology/attck-mapping.md)).
- Weakness classification: **CWE-79** (Cross-Site Scripting) and **OWASP Top 10 A03:2021** (injection).

**Investigation approach.** The analysis proceeded in phases: network analysis of the capture to find the exfiltration channel and collector, an application trace over the access log and request bodies to recover the injected payload, impact reconstruction (session hijack, anomalous account), and detection engineering. See [`notes/journal.md`](notes/journal.md).

**Evidence and timestamps.** The access log records UTC; the capture records CEST (UTC+2). Every timestamp in the deliverables is given in both to avoid misattribution.

---

## Tools used

- **tshark**: PCAP analysis and conversation/field extraction (collector identification, beacon flow, payload recovery).
- **Suricata 8.0.5**: rule validation by replaying the capture (`attack_eth.pcap`).
- **Base64 and hex-plus-URL decoding**: recovery of the exfiltrated beacon parameter and the injected payload from the request bodies.

---

## Detection engineering

Two rules, one per critical stage of the chain, each targeting the Suricata buffer suited to its intent:

| SID | Detects | Buffer | Match |
|---|---|---|---|
| 1000001 | XSS injection in feedback submission | http.request_body | `script` in POST body |
| 1000002 | Cookie exfiltration beacon | http.uri | `/collect` to 91.92.100.45:8080 |

Both rules fire against the incident capture for a total of 2 alerts (one injection, one beacon), covering the full attack chain with no false positives on the captured traffic. Full detail in `detection/README.md`.

---

## Repository layout

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
│   ├── INC-2026-006_Findings_Report.pdf   Full findings report (27 pages)
│   └── INC-2026-006_Findings_Report.md    Markdown source of the report, readable on GitHub
├── detection/
│   ├── local-xss.rules            The 2 Suricata rules (SID 1000001-1000002)
│   └── README.md                  Per-rule logic and validation
├── evidence-summary/
│   └── ioc-summary.md             Indicators of compromise, SIEM-ingestible format
├── methodology/
│   ├── attack-timeline.md         Full request-level timeline
│   └── attck-mapping.md           MITRE ATT&CK mapping table with section references
└── notes/
    └── journal.md                 Investigation journal (post-analysis reconstruction)
```

The findings report is provided as a 27-page PDF in `reports/`, alongside its Markdown source.

---

## Reproducibility

The evidence bundle is BeCode lab property and is not redistributed. With your own copy, the detection rules can be revalidated by replaying the capture offline:

```bash
rm -rf /tmp/suricata_test && mkdir /tmp/suricata_test
suricata -c <path>/suricata.yaml -S local-xss.rules -r attack_eth.pcap -l /tmp/suricata_test/
cat /tmp/suricata_test/fast.log
wc -l < /tmp/suricata_test/fast.log
```

The capture used is `attack_eth.pcap` (the Ethernet-framed version of the traffic). Both rules fire for a total of 2 alerts. Per-rule logic and validation detail are in [`detection/README.md`](detection/README.md).

---

## Known limits

- **Origin of `m.renard` undetermined.** The account authenticated from the attacker IP and matches no employee directory entry, but the evidence does not establish whether it was attacker-created or a pre-existing account that was compromised. Both readings lead to the same action: treat it as unauthorized and remove it.
- **Detection coverage is incident-scoped.** Rule 1000001 matches `script` in the feedback body and could fire on legitimate content that discusses scripting; rule 1000002 is tightly scoped to the known collector IP and port and would not catch a different collector. Both are documented in `detection/README.md`.
- **User agent is a soft indicator.** The Firefox-on-Linux user agent distinguishes attacker traffic from the Windows fleet, but a user agent can be spoofed: it supports attribution, it does not prove it.

---

## NexaCorp DFIR series

- [INC-2026-001](https://github.com/Jhatchi/NexaCorp-DFIR-INC-2026-001): Linux infrastructure compromise (vsftpd backdoor, Caldera C2)
- [INC-2026-002](https://github.com/Jhatchi/NexaCorp-DFIR-INC-2026-002): privilege escalation and persistence (Tor SSH, SUID, backdoor account)
- [INC-2026-003](https://github.com/Jhatchi/NexaCorp-DFIR-INC-2026-003): month-1 cross-incident assessment
- [INC-2026-004](https://github.com/Jhatchi/NexaCorp-DFIR-INC-2026-004): SQL injection (web portal)
- [INC-2026-005](https://github.com/Jhatchi/NexaCorp-DFIR-INC-2026-005): OS command injection and web shell (web portal)
- **INC-2026-006**: this repository
- [INC-2026-007](https://github.com/Jhatchi/NexaCorp-DFIR-INC-2026-007): IDOR and broken access control (NexaPortal); Month 2 capstone

---

## Acknowledgments

- **Thomas B.** (BeCode lab coach): scenario design and publication authorization for portfolio use.
- **MITRE** for the ATT&CK knowledge base used to map every finding.
- **Suricata project** for the detection engine used to validate the ruleset.

---

## About

Solo DFIR engagement delivered during the [BeCode Brussels](https://becode.org) Blue & Red Team bootcamp (November 2025 to September 2026), Mission 06.

Author: **[Johan-Emmanuel Hatchi](https://github.com/Jhatchi)** ([LinkedIn](https://www.linkedin.com/in/johan-emmanuel-hatchi/)).

Open to cybersecurity internship opportunities starting September 2026 in Belgium. Looking for SOC / DFIR / detection engineering roles where this kind of end-to-end work (PCAP forensics, log correlation, IDS rule writing, formal client reporting) is in scope.

---

## License

[MIT](LICENSE), 2026 Johan-Emmanuel Hatchi.

The report text and methodology notes are released under MIT: free to copy, adapt, and reuse with attribution. The evidence bundle, lab infrastructure, and original engagement briefings remain BeCode Brussels property and are not redistributed.
