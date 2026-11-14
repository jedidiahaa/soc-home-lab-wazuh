# Lessons learned

## 1. A slow VM can have a host-level cause

The Wazuh VM was painfully slow, and VirtualBox showed a turtle icon in its status bar. I timed 3,000 trivial commands inside the VM: 57.7 s at first, 29.7 s after I stopped WSL2 from running, and 3.97 s after I also turned off virtualization-based security in the registry and rebooted. The cause was Windows' own hypervisor running underneath VirtualBox. The cause had several layers, and each fix only removed part of it. The lesson: measure first, change one thing, measure again.

## 2. A frozen-looking install is not always frozen

The first Wazuh install seemed stuck for over 30 minutes. `ps` showed a process in state `T` (stopped), and sending it a continue signal resumed it. I never found out why it stopped. Later, a long pause turned out to be a package script running about 47,000 permission commands, which was slow but working. The first install then failed when the manager's start timed out, and the installer rolled itself back. The lesson: check process state, CPU use and the log before deciding something is stuck.

## 3. A 16 GB laptop decides the design

I measured memory use instead of guessing: 48% with both VMs off, 73% with the Wazuh server running, 90% with both running (95% at a later reading). That set the VM sizes (4096 MB for the server, 2560 MB for Windows) and the habit of running both VMs only while generating alerts.

## 4. Running on an unsupported OS version

Wazuh 4.14's install guide does not list Ubuntu 26.04. I installed with the OS check ignored (`-i`), and it worked for this lab. I would not do that in production.

## 5. Test activity has to produce the event you expect

A `net use` loop produced a single failed-logon event, not four. A different method (a secondary logon with a wrong password) produced ten failures from twelve attempts. I only knew because I counted the Windows events before and after (1, then 11). The dashboard also showed my agent as disconnected once while the agent's own log said it was online. A refresh fixed it. Check the source before blaming the tool.

## 6. What a correlation rule counts depends on one field

Built-in rule 60204 counts failed logons per source address. My rule 100100 counts them per target account. They catch different attacks (one source trying many accounts versus many sources trying one account). In my lab every local failure had the source `::1`, so the source-address grouping tells an analyst nothing here. I also saw that the failure that crosses a rule's threshold produces the correlation alert in place of its own single-event alert: 10 failures gave 9 alerts of one rule and 1 of the other.

## 7. Detection tools lag behind the ATT&CK matrix

The current ATT&CK ID for clearing Windows event logs is T1685.005, but Wazuh 4.14.9's database still uses the older T1070.001. With the new ID my alert carried no MITRE fields. I switched to the older ID and noted the change. Several built-in tags also fit their events badly (for example T1484 on a local group change).

## 8. A SIEM keeps what an attacker erases

After I cleared the Windows Security log, the alerts from my earlier tests were still in the Wazuh dashboard. That is the point of forwarding logs off the machine.

## What I would do next

Add process-creation logging (Sysmon) to answer "what started this?", turn on PowerShell logging for a suspicious-PowerShell detection, write a rule that links "account created" to "added to Administrators", and test the two-accounts experiment for 100100 versus 60204. See [detections/future-work.md](detections/future-work.md).
