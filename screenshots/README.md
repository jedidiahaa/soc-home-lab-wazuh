# Screenshots

Dashboard times are shown in Budapest time (UTC+1). The notes use UTC.

| File | What it shows |
|---|---|
| 01-agent-active.png | Wazuh Endpoints page: the Windows agent `win-victim` is active |
| 02-alert-list-60122-60204.png | Alerts from the first brute-force test: single failed-logon alerts (60122) and the built-in brute-force alert (60204) |
| 03a-alert-60204-eventdata.png | The 60204 alert, event fields (source `::1`, logon type 2, status codes) |
| 03b-alert-60204-rule.png | The same alert, rule fields (level 10, frequency 8, MITRE T1110) |
| 04-lockout-60115.png | The account lockout alert (60115, level 9) |
| 05-dashboard-overview.png | Dashboard overview with `win-victim` powered off to save memory, so the agent shows as disconnected |
| 06a-custom-rule-100100-list.png | Custom rule 100100 firing among the 60122 alerts and routine Windows background alerts |
| 06b-custom-rule-100100-rule.png | Rule 100100 details: level 8, frequency 5, MITRE T1110.001 |
| 07-vbox-settings-wazuh-server.png | `wazuh-server` VM settings. VirtualBox labels the OS "Ubuntu 25.04", its closest match. The installed OS is Ubuntu Server 26.04.1 |
| 08-vbox-settings-win-victim.png | `win-victim` VM settings |
| 10-vbox-manager-both-vms.png | VirtualBox Manager with both VMs |
| 11-snapshots-win-victim.png | Snapshots tab for `win-victim` (snapshot `win-agent-working`) |
| 12-new-admin-alert-list.png | Alerts after creating a test administrator account: 60109 (account created), 60154 (Administrators group changed, level 12) |
| 13-alert-60154-rule.png | The 60154 alert details. The built-in rule carries MITRE T1484 (Domain Policy Modification) for a local group change |
| 14-log-cleared-alert-list.png | The built-in alert 63103, "The audit log was cleared", level 5 |
| 15-log-cleared-100101-list.png | Two alerts from custom rule 100101 (level 12). The 13:53 one carries no MITRE fields (T1685.005 unknown to Wazuh). The 13:59 one is after switching to T1070.001 |
| 16-log-cleared-100101-rule.png | The 13:59 alert details: T1070.001, tactic Defense Evasion, technique Clear Windows Event Logs |
