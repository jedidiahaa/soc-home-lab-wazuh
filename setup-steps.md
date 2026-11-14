# Setup steps

Enough to rebuild the lab. Replace placeholders. Never write real passwords into this file.

## 1. Host preparation

- Hardware virtualization enabled (Task Manager, Performance, CPU: Virtualization Enabled).
- Install VirtualBox 7.2.22 from virtualbox.org. No Extension Pack.
- Windows hypervisor changes, needed because the VMs ran about 14 times slower with the hypervisor on (benchmark below). Run in an administrator PowerShell, then reboot once:

```
reg export "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard" "$env:USERPROFILE\deviceguard-backup.reg" /y
reg add "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard" /v EnableVirtualizationBasedSecurity /t REG_DWORD /d 0 /f
bcdedit /set hypervisorlaunchtype off
Disable-WindowsOptionalFeature -Online -FeatureName VirtualMachinePlatform -NoRestart
```

Also: Memory integrity turned off in Windows Security, and the Windows Features dialog used to untick Hyper-V, Windows Hypervisor Platform and Windows Subsystem for Linux (whichever were listed). Virtual Machine Platform was still enabled after the dialog, so I disabled it from PowerShell with `Disable-WindowsOptionalFeature`.

Check: `systeminfo | findstr /i hypervisor` printed nothing, and the VBS status was 0.

| Benchmark in the VM: `time (for i in $(seq 1 3000); do /bin/true; done)` | Result |
|---|---|
| Before any change (turtle icon) | 57.7 s |
| After WSL and features off | 29.7 s |
| After the VBS registry change and reboot | 3.97 s |

## Host changes to undo after the lab

1. Import `deviceguard-backup.reg` (or set `EnableVirtualizationBasedSecurity` back to 1).
2. `bcdedit /set hypervisorlaunchtype auto`
3. Tick Virtual Machine Platform and Windows Subsystem for Linux again.
4. Turn Memory integrity back on.
5. Reboot once, then run `systeminfo | findstr /i hypervisor` and `wsl -l -v` to confirm.

## 2. wazuh-server VM

- Ubuntu Server 26.04.1 LTS, install ISO checked with `certutil -hashfile <iso> SHA256`.
- VirtualBox: 4 vCPUs, RAM 4096 MB, 50 GB dynamic disk, no unattended install.
- Adapter 1 NAT, adapter 2 host-only. Installer choices: standard server (not minimized), guided storage on the whole disk (LVM, no encryption), OpenSSH server on, no snaps.
- Host-only address through netplan (`/etc/netplan/60-hostonly.yaml`, DHCP on the second interface with `use-routes: false`), then `sudo netplan apply`. Result: 192.168.56.101.

## 3. Wazuh 4.14.9

```
sudo mkdir -p /etc/systemd/system/wazuh-manager.service.d
printf '[Service]\nTimeoutStartSec=900\n' | sudo tee /etc/systemd/system/wazuh-manager.service.d/timeout.conf
sudo systemctl daemon-reload
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo nohup bash ./wazuh-install.sh -a -i > install.log 2>&1 < /dev/null &
```

- `-a` installs all components on one host. `-i` ignores the supported-OS check (needed on Ubuntu 26.04).
- The installer prints an admin password at the end. Store it privately. Never commit it.
- Afterwards, disable the Wazuh package repo so a routine upgrade cannot break it: `sudo sed -i "s/^deb /#deb /" /etc/apt/sources.list.d/wazuh.list`.

## 4. win-victim VM

- ISO: Windows 11 Enterprise LTSC 2024 evaluation (`26100.1742.240906-0331.ge_release_svc_refresh_CLIENT_LTSC_EVAL_x64FRE_en-us.iso`, about 4.99 GB) from Microsoft's Evaluation Center. 90-day evaluation.
- VirtualBox: EFI on, Secure Boot on, TPM 2.0, RAM 2560 MB, 2 CPUs, 64 GB dynamic disk. Adapter 1 NAT, adapter 2 host-only.
- Local account only (no Microsoft account). Time zone set to Central Europe Standard Time. Guest Additions installed for the shared clipboard.
- Address 192.168.56.102 on the host-only network. Ping to the server succeeded.

## 5. Wazuh agent

From the dashboard's "Deploy new agent" wizard: Windows, MSI, server 192.168.56.101, agent name `win-victim`. Run the generated command in an administrator PowerShell, then `NET START WazuhSvc`. Agent log: "Connected to the server ([192.168.56.101]:1514/tcp)". Dashboard: agent 001, status active.

## 6. Test activity (own VM only, throwaway account)

Run in an administrator PowerShell on win-victim. Create `testuser` with a throwaway password, and never target the real account:

```
net user testuser <throwaway-password> /add
$bad = ConvertTo-SecureString 'wrongpass' -AsPlainText -Force
$cred = New-Object PSCredential('testuser', $bad)
1..12 | ForEach-Object { try { Start-Process cmd.exe -Credential $cred } catch {}; Start-Sleep 1 }
```

Result: 4625 count went from 1 to 11 (10 failures from 12 attempts, likely because the account locked out after 10). Event 4740 (lockout) was recorded.

## Snapshots

| VM | Snapshot | State |
|---|---|---|
| wazuh-server | wazuh-installed | Wazuh installed, no agent |
| wazuh-server | rules-100100-100101-live | Both custom rules deployed |
| win-victim | win-agent-working | Agent connected |
| win-victim | tests-done | After all tests, powered off |

## Memory on a 16 GB laptop

| State | Memory use |
|---|---|
| Both VMs off | 48% (6.6 GB in use, 8.8 GB available at one reading) |
| wazuh-server running | 73% |
| Both VMs running | 90% (95% at a later reading) |

Figures as reported from Task Manager, with the VM sizes then in use (wazuh-server 4096 MB, win-victim 2560 MB).
