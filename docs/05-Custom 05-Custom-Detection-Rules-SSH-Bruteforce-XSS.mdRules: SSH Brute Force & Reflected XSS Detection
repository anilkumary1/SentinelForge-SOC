Custom Detection Rules: SSH Brute Force & Reflected XSS Detection
Objective

Build and tune custom Wazuh detection rules for two distinct attack categories: SSH brute-force login attempts (host-based, auth log source) and reflected Cross-Site Scripting against DVWA (web-based, Apache access log source). Validate each rule fires correctly and maps to the appropriate MITRE ATT&CK technique.

Lab Environment
Machine	OS	Purpose
Ubuntu 1	Ubuntu 24.04 LTS	Wazuh Manager (local_rules.xml lives here)
Ubuntu 2	Ubuntu 24.04 LTS	Wazuh Agent + DVWA (attack surface)
Part 1 — SSH Brute Force Detection
Step 1: First attempt — a rule that looked right but wasn't

The initial rule chained off Wazuh's default SSH auth-fail rule (5760) with no frequency threshold:

xml
<rule id="100020" level="12">
  <if_sid>5760</if_sid>
  <description>Custom: SSH authentication failed</description>
  <mitre><id>T1110.001</id></mitre>
  <group>authentication_failed,ssh,bruteforce,attack</group>
</rule>

Problem identified: this rule fired on every single failed login — including one-off typos — not on an actual burst pattern. Tagging a single failure as "bruteforce" (T1110.001) was inaccurate and would have flooded the dashboard with false positives.

Step 2: Corrected rule with frequency/timeframe correlation
xml
<rule id="100020" level="12" frequency="6" timeframe="120">
  <if_matched_sid>5760</if_matched_sid>
  <same_source_ip />
  <description>Custom: SSH brute force - 6 failed logins in 2 min</description>
  <mitre><id>T1110.001</id></mitre>
  <group>authentication_failed,ssh,bruteforce,attack,</group>
</rule>

frequency="6" timeframe="120" requires 6 matches from the same source IP within 120 seconds before this rule fires — this is what actually constitutes a brute-force pattern rather than isolated failures.

Step 3: Test

Manually failed SSH login 6+ times in under 2 minutes from a test source, then confirmed rule 100020 fired (not the underlying single-failure rule 5760 alone) in the Wazuh dashboard.

Part 2 — Reflected XSS Detection
Step 1: Rule definition
xml
<rule id="100012" level="8">
  <if_sid>31105</if_sid>
  <description>meduilm-HIGH: XSS attempt detected against monitored application</description>
  <group>attack,web,xss_attempt,</group>
</rule>

Chains off Wazuh's stock web-attack rule group (31105), which already includes generic XSS pattern matching against the Apache access log.

Step 2: Test

Submitted <script>alert('SentinelForge-Test')</script> into DVWA's Reflected XSS module at Low security. Confirmed:

JavaScript alert fired in-browser (proof the vulnerability executed)
Rule 100012 fired in Wazuh Discover, with the payload visible in the expanded alert's URL field
Step 3: Payload variation testing

Also tested <img src=x onerror=alert('XSS2')> and <svg onload=alert('XSS3')> to confirm the underlying decoder/rule wasn't only matching the literal <script> string but a broader XSS-pattern group.

Troubleshooting

Problem: Bruteforce rule fired on every single failure Cause: no frequency/timeframe/same_source_ip correlation on initial rule. Solution: added correlation attributes (Step 2 above).

Verification Checklist
 SSH bruteforce rule fires only on correlated burst, not single failures
 Bruteforce rule correctly tagged with T1110.001
 XSS rule fires on DVWA reflected XSS payload
 XSS rule confirmed against multiple payload variants
 Both rules visible in Wazuh Discover with correct rule ID and description
Lessons Learned
A rule that "fires" isn't automatically a correct rule — matching the pattern (burst behavior) rather than the symptom (a single failed login) is what separates a real detection from noise.
MITRE technique tags should reflect what the rule actually detects, not just where it's chained from — mislabeling a single-failure rule as T1110.001 would have been misleading in any audit of the ruleset.
Testing against payload variants (not just one canonical example) builds confidence that a rule generalizes rather than pattern-matching one specific string.
Next Phase

The next document is docs/06-SQL-Injection-Detection.md, which covers a considerably harder detection engineering problem: getting a custom SQL injection rule to fire at all, and the multi-stage debugging process required to find out why it initially didn't.
