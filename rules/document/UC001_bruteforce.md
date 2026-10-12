# UC-001 — Brute-Force Login Detection

## 1. Objective

Detect repeated failed SSH authentication attempts that may indicate a brute-force attack against a monitored (Windows) host.

## 2. Threat Scenario

An attacker repeatedly attempts to authenticate to a Windows host using different or guessed credentials.

If multiple authentication failures occur within a short period, the SOC analyst investigates whether the activity is malicious.

## 3. Detection Requirements

* **Trigger condition:** 5 failed SSH authentication events within 60 seconds.
* **Correlation key:** Source IP address.
* **Severity:** Level 10 — High (initial project setting; tune based on testing).
* **Platform:** Monitored Linux host.
* **Detection platform:** Wazuh SIEM.

## 4. Data Sources

* Wazuh agent log collection.
* Wazuh built-in SSH authentication rules.
* Windows desktop sign-in failures (These are a different event source. They can be used for a separate Windows brute-force detection, but only after configuring Windows Security event collection and identifying the relevant rules).

## 5. Detection Logic

1. Collect SSH authentication events from the monitored Linux host.
2. Identify failed authentication events using the existing Wazuh ruleset.
3. Correlate repeated failures from the same source IP within 60 seconds.
4. Generate a custom Wazuh alert when the threshold is reached.

## 6. MITRE ATT&CK Mapping

* T1110 — Brute Force.

## 7. Test Procedure

1. Confirm that the target Linux host is connected to Wazuh.
2. Generate five failed SSH login attempts from a test client within 60 seconds.
3. Confirm that Wazuh receives and recognizes each failure.
4. Check whether the custom correlation rule generates an alert.
5. Repeat with fewer than five failures as a negative test.

## 8. Expected Results

* Five qualifying failures within 60 seconds from the same source: custom alert generated.
* Fewer than five qualifying failures: no custom brute-force alert.
* Failures spread beyond the correlation window: no custom alert for that set of events, assuming no other qualifying events remain in the window.

## 9. Evidence to Collect

* Raw authentication event.
* Existing Wazuh rule ID.
* Custom rule XML.
* Alert JSON containing the custom rule ID.
* Test timestamps and source IP.
* Screenshots of positive and negative tests.

## 10. False Positives and Limitations

Legitimate users may repeatedly enter incorrect passwords. Automated monitoring systems may also generate repeated authentication failures.

The detection depends on authentication logs being collected and the source IP being decoded correctly. This use case detects a defined pattern; it does not prove that an attack succeeded.

## 11. Status

Pending implementation and validation.

## 12. Future Improvements

* Correlate attempts against multiple usernames.
* Detect distributed attempts from multiple source IPs.
* Add alert enrichment and analyst triage guidance.
* Tune the threshold based on observed lab activity.
