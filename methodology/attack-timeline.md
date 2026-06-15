# Attack Timeline: INC-2026-006

**Incident:** stored XSS and session hijacking on NexaPortal
**Date:** 18 June 2026

## Timezone note

The Apache access log (`web_access.log`) records time in UTC. The network capture (`attack.pcap`) records time in CEST (UTC plus two). To avoid confusion, every event below is given in both UTC and CEST.

## Detailed sequence

| # | Time (UTC) | Time (CEST) | Event | Evidence |
|---|-----------|-------------|-------|----------|
| 1 | 08:32:25 | 10:32:25 | Attacker 91.92.100.45 sends first request: GET /portal/ | web_access.log |
| 2 | 08:32:28 | 10:32:28 | Attacker requests /portal/login.php | web_access.log |
| 3 | 08:32:41 | 10:32:41 | Attacker POSTs to login.php (recon of auth) | web_access.log |
| 4 | 15:30:48 | 17:30:48 | Attacker POSTs XSS payload to /portal/feedback.php | web_access.log, attack.pcap |
| 5 | 15:30:52 | 17:30:52 | Second feedback POST from attacker | web_access.log |
| 6 | 15:45:54 | 17:45:54 | Victim p.dumont authenticates from 192.168.10.110 | web_access.log, attack.pcap |
| 7 | 15:50:58 | 17:50:58 | Victim loads feedback page, stored script executes | web_access.log |
| 8 | 15:51:00 | 17:51:00 | Beacon fires: cookie sent to 91.92.100.45:8080 | attack.pcap |
| 9 | 15:51:02 | 17:51:02 | Victim loads feedback page a second time | web_access.log |
| 10 | ~15:52 | ~17:52 | Attacker replays stolen session from 91.92.100.45 | web_access.log |

## Correction to the bundle README

The evidence bundle README stated that reconnaissance began around 17:30 CEST and that the beacon fired around 17:50:59 CEST. The evidence proves two refinements:

- First attacker contact was at 08:32:25 UTC (10:32:25 CEST), several hours earlier than reported. The compromise window is therefore wider than the README implied.
- The beacon fired at 15:51:00 UTC (17:51:00 CEST), confirmed by the network capture, with the triggering page load logged at 15:50:58 UTC.

These corrections are based on direct evidence and supersede the approximate values in the README.
