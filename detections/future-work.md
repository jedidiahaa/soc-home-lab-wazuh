# Future work (not done)

- Chain rule: account created, then added to Administrators within a few minutes. Needs experimenting with how Wazuh compares fields across two different event types.
- Suspicious PowerShell (encoded commands, T1059.001): needs script block logging (event 4104) turned on in Windows and the agent configured to collect that log.
- Switch rule 100101 back to T1685.005 once Wazuh's ATT&CK database knows the new IDs (the current Wazuh 4.14.9 does not).
- The 4 plus 4 experiment: 4 failed logons for each of two accounts. My prediction is that rule 60204 fires and rule 100100 does not.
- Process-creation logging (for example Sysmon), so an analyst can answer "what started this?".
