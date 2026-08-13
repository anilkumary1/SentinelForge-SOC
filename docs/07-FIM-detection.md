# File Integrity Monitoring: Real-Time Ransomware & Malware Drop Detection

## Objective
Configure Wazuh's File Integrity Monitoring (FIM/syscheck) module to detect file-level attack indicators in near real time: malware drops, webshell uploads, and ransomware-style mass file modification bursts. Validate detection using the industry-standard EICAR test file and a synthetic ransomware burst simulation.

## Lab Environment
| Machine | OS | Purpose |
|---|---|---|
| Ubuntu 1 | Ubuntu 24.04 | Wazuh Manager |
| Ubuntu 2 | Ubuntu 26.04  | Wazuh Agent (FIM monitored host) |

## Step 1 — Enable Realtime FIM
Edited the existing `<syscheck>` block in the agent's `ossec.conf` (critical: edited the existing block rather than adding a second one — see Troubleshooting):

```xml
<syscheck>
  <disabled>no</disabled>
  <frequency>21600</frequency>

  <directories realtime="yes" report_changes="yes">/var/www/html</directories>
  <directories realtime="yes">/tmp</directories>
  <directories realtime="yes">/home</directories>

  <ignore>/etc/mtab</ignore>
  <ignore>/etc/hosts.deny</ignore>
</syscheck>
```
`realtime="yes"` uses inotify on Linux, catching file changes within seconds rather than waiting for the default 6-hour scan cycle — essential for ransomware, where speed of detection matters.

## Step 2 — EICAR Test (Malware Drop Simulation)
EICAR is the industry-standard harmless test string recognized by every AV/scanner — safe to use, not real malware.

```bash
sudo curl -o /var/www/html/eicar_test.com https://secure.eicar.org/eicar.com.txt
```
(Note: this must run with `sudo` — `/var/www/html` is owned by `www-data`, and an unprivileged `curl` fails with a permission error.)

**Result:** a `syscheck` "file added" alert (rule group `syscheck_entry_added`, rule `554`) fired within seconds, with full forensic metadata:
```
syscheck.event: added
syscheck.path: /var/www/html/eicar_test.com
syscheck.sha256_after: [hash]
syscheck.md5_after: [hash]
syscheck.uid_after / gid_after / perm_after
```

## Step 3 — Ransomware Burst Simulation
```bash
for i in {1..50}; do echo "encrypted_content" >> /var/www/html/testfile$i.txt; done
```
Custom correlation rule to detect the burst pattern (not just individual file adds):
```xml
<rule id="100020b" level="14" frequency="20" timeframe="30">
  <if_matched_sid>554,550</if_matched_sid>
  <same_source_ip />
  <description>SentinelForge: Possible ransomware - mass file modification burst</description>
  <mitre><id>T1486</id></mitre>
  <group>ransomware,fim,</group>
</rule>
```
This fires on 20+ file add/modify events within a 30-second window — a behavioral signature of mass encryption, distinct from routine single-file activity.

## Troubleshooting

**Problem: Manager crashed after a syscheck edit — "exited" error, wouldn't restart**
Cause: a second, duplicate `<syscheck>` block was accidentally pasted into `ossec.conf` alongside the original default block, rather than editing the existing one. This produced malformed/duplicated XML content that failed to parse.

Diagnosis process:
```bash
xmllint --noout /var/ossec/etc/ossec.conf
```
(Critical detail: this must be run with `sudo` — without it, `xmllint` reports a generic "failed to load external entity" permission error that looks like an XML problem but is actually just a read-access failure, masking the real syntax error underneath.)

Solution: removed the entire malformed block and re-inserted one clean `<syscheck>` section:
```bash
sudo sed -i '/<syscheck>/,/<\/syscheck>/d' /var/ossec/etc/ossec.conf
```

**Problem: FIM alerts drowned in noise from normal desktop/browser activity**
Cause: `<directories realtime="yes">/home</directories>` monitors the *entire* home directory, including Firefox cache writes, `.config/dconf` files, and other routine desktop activity — none of which is attack-relevant, but all of which generates FIM alerts at the same severity as a genuine malware drop.

Solution: added exclusions for known-noisy paths:
```xml
<ignore type="sregex">.log$|.cache/mozilla|.mozilla/firefox|.local/share/gvfs-metadata</ignore>
```

## Verification Checklist
- [x] Realtime FIM confirmed active (inotify-based, not scan-interval-based)
- [x] EICAR test file triggers a `syscheck` "added" alert with full hash metadata
- [x] Ransomware burst simulation (50 files) triggers the dedicated burst-detection rule, distinct from individual file-add alerts
- [x] Noisy, non-attack-relevant paths excluded from FIM scope

## Lessons Learned
- **FIM configuration mistakes fail loudly (manager crash) rather than silently** — which is actually preferable for a lab environment, since a broken config is immediately obvious rather than quietly not detecting anything.
- **Broad directory monitoring without exclusions creates alert fatigue** — a genuinely important SOC skill demonstrated here is recognizing normal background noise (browser cache writes) versus real signal, and tuning scope accordingly rather than trying to triage every single alert manually.
- **Individual file-add detection and burst/ransomware detection are two different rules with two different purposes** — a single file add is routine; twenty file adds from the same source in thirty seconds is a behavioral anomaly. Building both, separately, gives layered coverage.

## Next Phase
The next document is `docs/08-Suricata-Integration.md`, covering network-layer intrusion detection — extending Sentinel Forge's visibility from the host layer (FIM) to the network layer (packet inspection, port scan detection).
