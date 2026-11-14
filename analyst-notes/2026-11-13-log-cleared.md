# Analyst note: Windows Security log cleared

## Summary

- Alert: custom rule 100101, level 12, "Windows audit log cleared: possible attacker covering tracks". MITRE T1070.001 (Clear Windows Event Logs), tactic Defense Evasion. It builds on built-in rule 63103 (level 5).
- Host: win-victim.
- Time: about 12:59:19 UTC (alert 13:59:20 Budapest time). The underlying event is Windows event 1102.

## What happened

The Security event log on `win-victim` was cleared. Windows writes event 1102 ("the audit log was cleared") into the newly emptied log, and the agent forwarded it. The same thing happened earlier at 12:20:57 UTC (alert 63103, level 5) and at 12:53:39 UTC (first 100101 alert).

## How I confirmed it

- Read the alert fields: event ID 1102, provider Microsoft-Windows-Eventlog, channel Security.
- The event names who cleared the log: `data.win.logFileCleared.subjectUserName` is `labadmin` (domain DESKTOP-I4CMLJC, logon ID 0x2d2b4). It also records the ID of the process that did it, `clientProcessId` 4536, but not the process name.
- I ran the command myself (`wevtutil cl Security`) just before each alert.
- The alerts from my earlier tests were still visible in the Wazuh dashboard after the local log was wiped. That is the point of forwarding logs to a SIEM: the attacker's copy can be erased, the SIEM's copy stays.

## Verdict

True positive, benign. I cleared the log myself as a test on my own lab VM. Without that knowledge it is a strong sign of someone hiding tracks, which is why I raised it from level 5 to level 12.

## What I would do next if this were real

1. Check who cleared it (`labadmin` here) and whether that account was expected to do this. Look up the logon session (logon ID 0x2d2b4) to see how and from where that account logged on.
2. Review what happened before the clearing, using the SIEM's copy of the logs, since the local ones are gone.
3. Check for other alerts on this host around the same time: new accounts, group changes, logons, anything at level 8 or above.
4. If it was not authorised: isolate the host, preserve what remains, and escalate.

## Known limits

- Administrators sometimes clear logs on purpose, so this rule needs a quick human check, and it will fire on legitimate maintenance.
- The lab has no process-creation logging, so I cannot say which program cleared the log. The event gives only the process ID (4536), which is not enough on its own.

## Evidence

- `screenshots/15-log-cleared-100101-list.png`
- `screenshots/16-log-cleared-100101-rule.png`
