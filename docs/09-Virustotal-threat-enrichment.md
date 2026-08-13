# VirusTotal Threat Enrichment: An Honest Debugging Postmortem

## Objective
Integrate Wazuh's built-in VirusTotal module to automatically check file hashes flagged by FIM against VirusTotal's reputation database — enriching a raw "file added" event into "file added AND confirmed malicious." This document is included deliberately despite the integration not reaching a fully confirmed working state, because the debugging process eliminated a long list of plausible causes methodically, which is itself a demonstrable and transferable skill.

## Lab Environment
| Machine | OS | Purpose |
|---|---|---|
| Ubuntu 1 | Ubuntu 24.04  | Wazuh Manager (integration runs here) |
| Ubuntu 2 | Ubuntu 26.04 | Wazuh Agent + DVWA (FIM source) |
| Kali Linux | Kali Rolling | Attacker machine (not directly involved in this integration) |

## Configuration
```xml
<integration>
  <name>virustotal</name>
  <api_key>[REDACTED]</api_key>
  <group>syscheck</group>
  <alert_format>json</alert_format>
</integration>
```

## The Debugging Sequence

### Issue 1: Persistent 403 "Check credentials" errors
Every syscheck event (including routine, non-malicious file writes) triggered a VirusTotal lookup attempt, and every attempt returned:
```
data.virustotal.error: 403
rule.description: VirusTotal: Error: Check credentials
```
This fired continuously — hundreds of times per hour — because FIM was scoped broadly across `/home`, and every Firefox cache write or config change triggered a fresh lookup attempt.

**Diagnosis:** verified the API key directly against VirusTotal's API, bypassing Wazuh entirely:
```bash
curl --request GET --url 'https://www.virustotal.com/api/v3/users/current' --header 'x-apikey: [KEY]'
```
This succeeded, confirming the key itself was valid — the problem was specifically in how Wazuh was using it.

**Root cause found:** on close comparison, the key stored in `ossec.conf` had a duplicated character sequence in the middle (`...724c2e078c**42e078c**4f340958e` instead of `...724c2e078c4f340958e`) — introduced during a retyping/re-paste cycle across multiple edit attempts. An invalid key produces exactly a 403, and the corruption was easy to miss by eye since the start and end of the key looked correct.

**Fix:** retyped the key fresh in a single clean edit rather than incrementally correcting it.

### Issue 2: `Exit status 4` after the credential fix
With a corrected key, the 403 error changed to a different failure:
```
wazuh-integratord: ERROR: Unable to run integration for virustotal
wazuh-integratord: ERROR: Exit status was: 4
```
`integrations.log` (the log specifically meant to capture the integration script's own output) remained empty throughout, which itself became a diagnostic dead-end for some time.

**Diagnosis path:**
1. Confirmed the integration script (`virustotal.py`) existed with correct permissions (`-rwxr-x---`, owned `root:wazuh`)
2. Confirmed Wazuh's bundled Python interpreter (`/var/ossec/framework/python/bin/python3`) launched correctly
3. Ran the script manually, replicating exactly what `wazuh-integratord` does internally, to surface the real error instead of the generic wrapper-level exit code:
```bash
sudo /var/ossec/framework/python/bin/python3 /var/ossec/integrations/virustotal.py /tmp/test_alert.json [API_KEY] null
```
This produced an actual Python traceback for the first time:
```
File "virustotal.py", line 116, in process_args
    raise Exception
```

**Root cause:** the script's `process_args()` raises this generic exception whenever `request_virustotal_info()` returns an empty response — which happens whenever VirusTotal has no record of the submitted file hash. The first manual test used a synthetic, invalid 6-character placeholder hash instead of a real SHA256, which VirusTotal correctly rejected.

**Retest with a real hash** (EICAR's known SHA256) still returned the same exception — narrowing the remaining possibilities to either genuine API rate-limiting (the free tier allows 4 lookups/minute and 500/day, and the earlier 403 retry storm had made hundreds of rapid-fire attempts, plausibly exhausting the quota) or a subtler issue in the request-construction logic not yet isolated.

## Decision: Deprioritized, Not Abandoned
At this point, further debugging would have required either waiting out a potential rate-limit window or adding verbose HTTP-level debugging inside the script itself. Given that:
- FIM (the core detection mechanism VirusTotal was meant to *enrich*, not replace) was already confirmed fully functional independently
- VirusTotal integration is an enhancement layer, not a required component of file-integrity detection
- Continued debugging had reached a point of diminishing returns relative to time invested

...the decision was made to document the state honestly and move forward with other detection categories, with VirusTotal integration left as a documented "next step" rather than a claimed completed feature.

## Verification Checklist
- [x] API key validity confirmed independently via direct `curl` against VirusTotal's API
- [x] Key corruption identified and corrected in `ossec.conf`
- [x] Integration script confirmed present, executable, correct ownership
- [x] Python interpreter confirmed functional
- [x] Manual script execution reproduced the real error (not just the generic wrapper exit code)
- [ ] End-to-end automated lookup via `wazuh-integratord` confirmed working — **not yet resolved**

## Lessons Learned
- **A single visually-similar character error in a long API key can hide in plain sight** — visual inspection of "does the start and end look right" is not sufficient; a byte-for-byte fresh retype resolved what looked like a much deeper credentials problem.
- **Generic exit codes (like "exit status 4") often mask a specific, findable error one layer down** — the discipline of manually reproducing what a wrapper/daemon does internally, rather than treating the wrapper's own error as final, is what actually surfaces the real cause.
- **Retry storms compound problems** — hundreds of rapid failed attempts (from the initial credential bug, before it was fixed) likely consumed rate-limit quota that then obscured whether the *fixed* credentials were actually working cleanly afterward. Broad FIM scope without exclusions directly contributed to this by generating far more lookup attempts than were ever necessary.
- **Knowing when to stop is itself a skill.** Continuing to debug a non-critical enrichment feature indefinitely, at the expense of building out the remaining detection categories, would have been a worse use of project time than documenting the state honestly and moving on.

## Next Phase
With core host-based (FIM), web-application (SQLi/XSS), authentication (bruteforce), and network-based (Suricata) detection confirmed working, the next phase shifts to expanding coverage into the remaining planned categories: webshell detection, privilege escalation, and anonymous/VPN login detection — followed by SOAR case-management integration via TheHive.
