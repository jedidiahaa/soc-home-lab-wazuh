# Lab log

Times are UTC unless stated. One entry per session. Edit anything that does not match what you did.

## 2026-10-30

**Goal:** get a Wazuh server running.

**What I did**
- Installed VirtualBox 7.2.22 (ignored the optional Python bindings warning).
- Downloaded the Ubuntu Server 26.04.1 ISO (2.72 GB in Windows). The SHA256 matched the published value: cc8a95cde20f6ced61a322420de00f10cc3c90ced545daa46cb9c1a117f1d927.
- Built the `wazuh-server` VM (Ubuntu Server, 50 GB dynamic disk, OpenSSH on). Added a second, host-only adapter and got SSH working from Windows at 192.168.56.101.
- Started the Wazuh 4.14 installer at about 23:21 with `-a -i` (all components, OS check ignored).

**What went wrong:** the install appeared stuck for 30+ minutes at the dependencies step. `ps` showed the apt process in state `T` (stopped). Sending `SIGCONT` resumed it. Root cause not determined.

## 2026-10-31

**Goal:** finish the Wazuh install, add a Windows victim, see the first alerts.

**What went wrong:** the first install failed at 00:36. The manager's start operation timed out (about 45 s), and the installer removed everything. The permissions script in the manager package (about 47,000 lines) took over 15 minutes.

**Diagnosis:** the VM was very slow. The VirtualBox status bar showed a turtle icon, which means it was running on top of Windows' own hypervisor. A benchmark of 3,000 `/bin/true` calls took 57.7 s.

**Fix:** turned off the Windows features and settings that keep the hypervisor on (full list in setup-steps.md). Benchmark: 57.7 s, then 29.7 s after WSL stopped running, then 3.97 s after the virtualization-based security registry change and a reboot.

**Result:** reinstall of Wazuh 4.14.9 took about 18 minutes (summary printed 01:41). Dashboard reachable at https://192.168.56.101 (self-signed certificate warning is expected).

**Windows VM:** created `win-victim` (Windows 11 Enterprise LTSC 2024 evaluation), local account, second host-only adapter at 192.168.56.102. Memory was the limit on a 16 GB laptop (see lessons-learned.md). Installed the Wazuh agent 4.14.9 from the dashboard's deploy wizard. The agent log reported "Connected to the server" and the dashboard showed it active (after refreshing a stale page).

**First detections:** a `net use` loop produced only one failed logon event. A script using secondary logon (`Start-Process -Credential`) against a throwaway account produced 10 failures from 12 attempts and a lockout. Built-in rules 60122, 60204 and 60115 fired.

**Evidence:** screenshots 01 to 03, the 60204 alert JSON, installer output.

**Next:** write and test custom rule 100100, then the other detections.

## 2026-11-13

**Goal:** check the lab still works after a break of a couple of days, and tidy the evidence.

**What I did**
- Started `wazuh-server` and opened the dashboard. It said "Wazuh dashboard server is not ready yet" for a few minutes (the dashboard waits for the indexer to finish starting), then loaded and I logged in.
- Re-took the screenshots (10 in total, see screenshots/README.md). The alerts from 31 October were still stored, so no test had to be re-run.
- Took a snapshot `win-agent-working` of `win-victim` while it was powered off. The server's latest snapshot is `wazuh-installed`.
- Read-only look at the server's custom rules file: `/var/ossec/etc/rules/local_rules.xml` has 19 lines and one example rule (ID 100001), so ID 100100 is free. Made a backup copy at `~/local_rules.xml.bak`. `wazuh-logtest` is installed (needs `sudo`).

**What I noticed**
- With `win-victim` powered off, the overview showed the agent as disconnected, 20 medium and 2 low alerts in the last 24 hours. I think the alerts come from the server's own logs, but I have not checked.
- My 31 October burst logged 10 failed logons in Windows. The alert list shows 9 alerts for rule 60122 plus one for rule 60204, so 10 alerts for 10 events. This is consistent with the event that crossed the threshold (the 8th) producing the correlation alert 60204 instead of a plain 60122. I worked this out from the alert counts, not from reading the rule engine's code.

**What I learned:** custom Wazuh rule IDs start at 100000, and the dashboard's "not ready" message right after boot is just the indexer still starting.

**Next:** deploy custom rule 100100 and test it against the throwaway account.

**Later the same day: custom rule 100100**
- Added rule 100100 (5 failed logons for the same account within 120 seconds, T1110.001) to `local_rules.xml`. Checked syntax with `wazuh-analysisd -t` (exit 0), restarted the manager, and the agent reconnected.
- Before testing I predicted what would fire. Then I ran 6 failed logons against the throwaway account on `win-victim`.
- Result: four 60122 alerts for failures 1 to 4, then 100100 at 12:39:01 (Budapest time) for the 5th, then a plain 60122 at 12:39:02 for the 6th. 60204 did not fire. This matched my prediction for the correlation rules.
- The same list also showed routine Windows background alerts (services starting, CIS benchmark checks). I have not opened them. They are useful for telling noise from real alerts.
- Learned: `same_field` decides what the rule treats as one attacker. 60204 groups by source address and 100100 groups by account, so they catch different patterns.

**Later the same day: new administrator account**
- Searched the built-in rules for events 4720 and 4732 first. Rules for account creation (60109) and group changes already exist.
- Created the throwaway account `labtest2` and added it to Administrators on `win-victim`. Alerts: 60109 (level 8) at 13:03:13 Budapest time, then 60154 (level 12) at 13:03:27.
- Decided not to write a custom rule for this: both halves are already covered by built-in alerts. The missing piece is a rule linking the two steps, listed as future work.
- Checked the MITRE tags on the built-in rules against attack.mitre.org. They do not describe these events well.

**Later the same day: Windows log cleared**
- Cleared the Security log on `win-victim` with `wevtutil cl Security`. Built-in rule 63103 ("The audit log was cleared") fired at level 5 at 13:20:57.
- Noticed the level seems low for a defense-evasion event, compared with level 12 for an Administrators group change. This is the argument for a small custom rule that raises it.
- A burst of group-change alerts also appeared at 13:20:13. I think it came from deleting the test admin account (removal from its groups). The account was `labtest2`. I did not check the event ID in that 60154 alert.
- Drafted rule 100101 myself (level 12, child of 63103, ATT&CK T1685.005). I had suggested T1070.001, the older ID. T1685.005 is the current ID for Clear Windows Event Logs.
- Deployed it (syntax check, manager restart) and cleared the log again. Rule 100101 fired at 13:53:39 at level 12, and the old 63103 alert did not repeat. The alert had no MITRE fields, probably because Wazuh's database predates the new ID. Wazuh's own rule 63103 uses the older T1070, so I switched my rule to the older sub-technique T1070.001, restarted the manager and cleared the log again. The 13:59 alert showed id T1070.001, tactic Defense Evasion, technique Clear Windows Event Logs. Resolved.

**End of the lab work, 2026-11-13**
- Powered off both VMs cleanly and took snapshots: `rules-100100-100101-live` (wazuh-server) and `tests-done` (win-victim).
- Final scope: 3 attack scenarios (failed logons, new admin account, cleared log), 2 custom rules (100100, 100101), 3 analyst notes.
- Next: resolve the remaining notes, publish the repo, undo the host changes.
