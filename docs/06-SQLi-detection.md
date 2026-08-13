# SQL Injection Detection: Debugging the Decoder-to-Rule Pipeline

## Objective
Build a custom Wazuh rule to detect SQL injection attempts against DVWA. This document deliberately documents the full debugging journey — including three separate root causes that had to be found and fixed in sequence — because the troubleshooting process demonstrated more practical SOC/detection-engineering skill than a clean first-try success would have.

## Lab Environment
| Machine | OS | Purpose |
|---|---|---|
| Ubuntu 1 | Ubuntu 24.04 LTS | Wazuh Manager |
| Ubuntu 2 | Ubuntu 24.04 LTS | Wazuh Agent + DVWA |

## The Rule (final, working version)
```xml
<rule id="100014" level="10">
  <if_sid>31100</if_sid>
  <decoded_as>web-accesslog</decoded_as>
  <url type="pcre2">(?i)('|%27).{0,15}(or|and).{0,15}(=|%3D)|union(\s|\+%20)+select|sleep\(|benchmark\(|information_schema</url>
  <description>SQL Injection attempt detected in web request</description>
  <mitre><id>T1190</id></mitre>
  <group>web,sqli,attack,</group>
</rule>
```

This regex catches quote-based injection (`' OR '1'='1`), UNION-based enumeration, blind SQLi via `sleep()`/`benchmark()`, and `information_schema` reconnaissance — all URL-encoded or raw.

## The Debugging Journey

### Root Cause #1: Wrong log source path entirely
The initial `ossec.conf` `<localfile>` block pointed at a path that didn't exist:
```xml
<location>/var/log/dvwa/access.log</location>
```
This looked plausible but was never actually created — DVWA is served *by* Apache directly, not through a separate log directory. Real traffic was landing in `/var/log/apache2/access.log`, which Wazuh was never watching. **Result:** zero data ever reached the rule engine, regardless of how correct the rule itself was.

**Fix:** corrected the `<location>` to the real Apache log path, restarted the agent.

### Root Cause #2: Manual test line had a formatting error
While validating with `wazuh-logtest`, an early test used a hand-retyped log line with `--` (no space) instead of the correct Apache combined-log-format `- -` (space-separated double dash) between the client IP and timestamp. This single missing space broke the decoder's regex match and produced a misleading "No decoder matched" result — which looked like a real bug but was actually bad test input.

**Fix:** used freshly copied real log lines from `tail` output instead of retyping them by hand.

### Root Cause #3: Missing rule chaining (`<if_sid>`)
Once decoding was confirmed working (`wazuh-logtest` correctly parsed `decoded_as: web-accesslog`, extracted `url`, `srcip`, etc.), the rule still didn't fire — Phase 3 of `wazuh-logtest` showed only the generic base rule `31100` ("Access log messages grouped") matching, never the custom rule `100014`.

**Root cause:** the custom rule had no `<if_sid>` — without chaining it as a child of the base access-log rule, Wazuh's rule engine never evaluated it against the same pipeline pass that claimed the match.

**Fix:** added `<if_sid>31100</if_sid>` to rule `100014`, matching the pattern used throughout Wazuh's own default ruleset.

## Final Verification
1. Ran `1' UNION SELECT user, password FROM users-- -` against DVWA's SQLi module at Low security — page returned dumped usernames/password hashes, confirming exploit success
2. Confirmed the payload landed in `/var/log/apache2/access.log`
3. Fed the real log line into `wazuh-logtest` — Phase 3 now showed `id: 100014` instead of `31100`
4. Confirmed the alert appeared in Wazuh Discover with:
   - `rule.id: 100014`
   - `rule.description: SQL Injection attempt detected in web request`
   - `rule.mitre.id: T1190`
   - `data.url` containing the full injected payload

## Troubleshooting Summary Table
| Symptom | Root Cause | Fix |
|---|---|---|
| No events at all in the log | `<location>` pointed at nonexistent path | Corrected to real Apache log path |
| `wazuh-logtest` said "No decoder matched" | Malformed manual test input (missing space) | Used real copied log lines, not retyped ones |
| Decoder worked, but only base rule `31100` matched | Missing `<if_sid>` chaining on custom rule | Added `<if_sid>31100</if_sid>` |

## Verification Checklist
- [x] Apache access log path correctly configured in `ossec.conf`
- [x] `wazuh-logtest` confirms `decoded_as: web-accesslog` on real traffic
- [x] Custom rule `100014` fires (not just base rule `31100`)
- [x] Confirmed against live DVWA SQLi payload, not just synthetic test input
- [x] Alert visible in Wazuh Discover with correct MITRE mapping (T1190)

## Lessons Learned
- **A "no match" result from a diagnostic tool doesn't always mean the config is wrong — verify the input to the diagnostic tool is itself correct first.** The manual retyping error cost real debugging time chasing a phantom decoder problem.
- **Decoding and rule matching are two separate pipeline stages.** A correctly decoded log with all fields extracted can still fail to produce an alert if rule chaining (`<if_sid>`) isn't set up — this is a genuinely easy trap to fall into when writing rules from scratch rather than modifying existing ones.
- **Always validate against the file path actually receiving traffic**, not the path that seems like it should. A silently-wrong log path is one of the hardest bugs to catch, because everything else in the config can be perfectly correct.

## Next Phase
The next document is `docs/07-FIM-Detection.md`, covering File Integrity Monitoring setup for ransomware and malware-drop detection, including the EICAR test-file validation methodology.
