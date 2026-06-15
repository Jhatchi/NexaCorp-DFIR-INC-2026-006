# MITRE ATT&CK Mapping: INC-2026-006

**Incident:** stored XSS and session hijacking on NexaPortal
**Framework:** MITRE ATT&CK for Enterprise

This mapping links each observed action to its ATT&CK technique, with the evidence that supports it. Techniques are listed in the order they occur in the attack chain.

| Phase | Technique ID | Technique | Observed action | Evidence |
|-------|--------------|-----------|-----------------|----------|
| Reconnaissance | T1595.003 | Active Scanning: Wordlist Scanning | Attacker probes /portal/ and /portal/login.php before the attack | web_access.log |
| Initial Access | T1190 | Exploit Public-Facing Application | Attacker exploits the unsanitized feedback feature of NexaPortal | web_access.log, attack.pcap |
| Execution | T1059.007 | Command and Scripting Interpreter: JavaScript | Injected `<script>` runs in the victim browser on page load | attack.pcap |
| Persistence (candidate) | T1136.001 | Create Account: Local Account | Possible creation of the `m.renard` account (origin undetermined) | attack.pcap |
| Defense Evasion | T1027 | Obfuscated Files or Information | Cookie value is Base64-encoded with btoa before exfiltration | attack.pcap |
| Credential Access | T1539 | Steal Web Session Cookie | Script reads document.cookie and exfiltrates the session id | attack.pcap |
| Collection | T1119 | Automated Collection | Script automatically harvests the cookie when the page loads | attack.pcap |
| Exfiltration | T1041 | Exfiltration Over C2 Channel | Cookie sent to attacker collector via HTTP beacon | attack.pcap |
| Lateral Movement | T1550.004 | Use Alternate Authentication Material: Web Session Cookie | Attacker replays the stolen PHPSESSID to access the portal | web_access.log |

## Notes on technique selection

- **T1136.001 is a candidate, not a confirmed technique.** The `m.renard` account may have been created by the attacker, or it may be a pre-existing account that was reused. The mapping flags it as a candidate to stay consistent with the rest of the report, which does not assert which hypothesis holds.
- **T1027 (obfuscation) applies to the btoa Base64 step.** This is light obfuscation, intended to make the exfiltrated value less obvious in transit, not strong encryption.
- **T1550.004 is the core impact.** The whole attack chain exists to reach this technique: authenticating as the victim without ever obtaining the password.
