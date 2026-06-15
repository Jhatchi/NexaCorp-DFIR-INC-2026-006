# Detection Engineering: INC-2026-006

This directory contains the Suricata rules written to detect the INC-2026-006 attack chain, and the validation results.

## Files

- `lab.rules` : the two Suricata rules (SID 1000001 and 1000002).

## Validation method

Each rule was validated by replaying the incident capture offline:

```bash
rm -rf /tmp/suricata_test && mkdir /tmp/suricata_test
suricata -c <path>/suricata.yaml -S lab.rules -r attack_eth.pcap -l /tmp/suricata_test/
cat /tmp/suricata_test/fast.log
wc -l < /tmp/suricata_test/fast.log
```

The capture used was `attack_eth.pcap`, the Ethernet-framed version of the incident traffic. The flag for each detection is the number of alerts the rule produces.

## Rules and results

### Rule 1: XSS injection in feedback POST (SID 1000001)

```
alert http any any -> any any (msg:"NEXACORP XSS injection in feedback POST"; flow:to_server,established; http.request_body; content:"script"; nocase; sid:1000001; rev:1;)
```

Detection logic: inspect the HTTP request body for the `script` marker. A legitimate feedback submission contains plain text; the malicious one contains a script tag. The `nocase` modifier makes the match case-insensitive.

Result: **1 alert.** The capture contains a single injecting POST that carries the payload.

### Rule 2: Cookie exfiltration beacon to collector (SID 1000002)

```
alert http any any -> 91.92.100.45 8080 (msg:"NEXACORP cookie exfiltration beacon to collector"; flow:to_server,established; http.uri; content:"/collect"; sid:1000002; rev:1;)
```

Detection logic: target outbound HTTP traffic to the collector IP and port, restricted to the `/collect` endpoint. The `http.uri` sticky buffer is placed before its `content`, as required by Suricata rule syntax.

Result: **1 alert.** One beacon fired from the victim workstation to the collector.

### Full coverage

With both rules active against the capture, the total is **2 alerts** (one injection, one beacon). The set covers the complete attack chain with no false positives on the captured traffic.

## False positive considerations

- Rule 1 matches on `script` in the request body. In a production environment, legitimate content (for example a feedback entry that genuinely discusses scripting) could trigger it. For production use, tighten the match (for example require the `<` opening byte or `%3Cscript`, or combine with the feedback URI) to reduce noise.
- Rule 2 is tightly scoped to a known collector IP and port. It is precise for this incident but would not catch a future attack using a different collector. For broader coverage, a behavioral rule on outbound POSTs carrying Base64-looking parameters to non-standard ports would generalize better, at the cost of more false positives.

## Note on environment

The rules were validated with Suricata 8.0.5. The sticky buffer ordering used here (`http.uri` before `content`, `http.request_body` before `content`) is valid in both Suricata 6.x and 8.x.
