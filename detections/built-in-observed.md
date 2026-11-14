# Built-in Wazuh rules that fired

These are Wazuh's own rules. They are not mine, and the README must say so.

Source file on the server: `/var/ossec/ruleset/rules/0580-win-security_rules.xml`.

## 60122 - Logon Failure - Unknown user or bad password

- Level 5. Fires on Windows event 4625 (or 529). Group `authentication_failed`.
- MITRE tag in the ruleset: T1531 (Account Access Removal). I think T1110 (Brute Force) would be a closer fit for a plain failed login.
- Observed: one alert per failed logon, except the failure that crossed the 60204 threshold, which produced a 60204 alert instead (10 failures gave 9 alerts of 60122 and 1 of 60204).

## 60204 - Multiple Windows Logon Failures (the built-in brute-force rule)

- Level 10, MITRE T1110 (Brute Force), tactic Credential Access.
- Logic: counts alerts in the `authentication_failed` group. `frequency="$MS_FREQ"` with `timeframe="240"`. `$MS_FREQ` is defined as 8 in `0585-win-application_rules.xml`. So 8 failures within 240 seconds.
- It requires the same `win.eventdata.ipAddress` across the failures.
- Observed: fired at 12:37:48 UTC on the eighth failure of my test burst.
- Limitation I noticed: for logons started on the machine itself the address was `::1`, so every local failure shares the same value. The rule groups by source address, which says nothing about which account was attacked.

## 60115 - User account locked out (multiple login errors)

- Level 9. Fired at 13:37:50 Budapest time (12:37:50 UTC), right after the burst. Windows recorded event 4740 for the lockout.
- I did not open this rule's definition.

## How I tested

Test activity is described in setup-steps.md, section 6 (throwaway account, own VM only).

## New local account added to Administrators (tested 2026-11-13)

Test: on `win-victim` I created the throwaway account `labtest2` and added it to the local Administrators group (two `net` commands, own VM only).

| Rule | Level | Description | Note |
|---|---|---|---|
| 60109 (two alerts) | 8 | User account enabled or created | Covers events 4720 and 4722 (the rule file lists both) |
| 60110 | 8 | User account changed | |
| 60160, 60170 | 5 | Domain Users / Users Group Changed | Probably from Windows placing a new account in default groups. I did not open these alerts. |
| 60154 | 12 | Administrators Group Changed | Fired 14 seconds after the account creation. MITRE tag shown: T1484 |

- The group-change alert names who did it (`subjectUserName` = `labadmin`) and the group (`targetUserName` = `Administrators`). The new account appears as a member SID, not by name.
- MITRE tags: the built-in rules tag the group change as T1484 (domain or tenant policy modification), and tag 60122 as T1531 (account access removal). I checked attack.mitre.org and neither describes these events well. Closer candidates to look up: T1098 (Account Manipulation) for the group change, T1136.001 (Create Account: Local Account) for the creation.
- Coverage gap: no rule ties the two steps together (account created, then made administrator within minutes). A chain rule would probably need to match any creation followed by any Administrators change on the same host, because the two events carry the account in different fields. Not written. See future-work.md.

## Windows Security log cleared (tested 2026-11-13)

Test: ran `wevtutil cl Security` in an administrator PowerShell on `win-victim` (own VM, throwaway lab).

- Rule 63103, level 5, "The audit log was cleared". Fired at 13:20:57 Budapest time (12:20:57 UTC).
- MITRE tag on the rule: T1070 (Indicator Removal), the older numbering. The relevant technique is Clear Windows Event Logs. In the current ATT&CK matrix it is T1685.005 (Disable or Modify Tools: Clear Windows Event Logs). Before the April 2026 renumbering it was T1070.001, which older tools and databases still use.
- Observation: level 5 looks low for log clearing, which is a common way to hide tracks after an intrusion. For comparison, the built-in Administrators group change is level 12 and the brute-force rule is level 10. An alert at level 5 could be filtered out of an analyst's queue.
- Idea: a small custom rule that raises the severity of 63103. See custom-100101.md.
- The same list also showed another burst of group-change alerts (60154, 60160, 60170) at 13:20:13. I think it came from deleting `labtest2`, which removes it from its groups. A "User account disabled" alert appeared in the same cluster. I did not check the event ID in that 60154 alert.
