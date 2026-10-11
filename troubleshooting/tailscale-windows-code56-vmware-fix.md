# Tailscale Tunnel Code 56 on Windows: VMware Bridge notifier workaround

## Summary

On one Windows machine, the Tailscale Tunnel network adapter remained in Device Manager with **Code 56** (`CM_PROB_NEED_CLASS_CONFIG`) even after reinstalling Tailscale and restarting Windows. The Tailscale service was running and a signed Wintun driver package was installed, but Windows had not configured the virtual adapter successfully.

Exporting and deleting one specific VMware Bridge notifier CLSID registry key, then restarting Windows, resolved the adapter error on this machine. Tailscale subsequently authenticated and returned the tailnet device list.

**Important:** This is a user-confirmed workaround on one machine, not a universal fix or official recommendation from Tailscale or VMware. The underlying cause has not been proven.

## Environment

- System profile: Windows 11 Pro, build `26300.9550`. Note: `Get-ComputerInfo` reported `Windows 10 Pro / Version 2009` for the same build, so the product-name reporting was inconsistent.
- Tailscale installer: `1.104.1` (Windows x64 MSI)
- Adapter: `Tailscale Tunnel`, device instance beginning `SWD\\WINTUN\\`
- Error before the fix: `56 (0x38) [CM_PROB_NEED_CLASS_CONFIG]`
- Driver provider: WireGuard LLC
- Driver package: `oem64.inf`, Wintun driver version `0.14.0.0`, signed
- `netcfg -s n` listed the `vmware_bridge` networking service.

## Symptoms and diagnostics

- The Tailscale Windows service was running, but the Tailscale Tunnel adapter had Code 56.
- Wi-Fi and Ethernet adapters did not report Plug and Play errors.
- `NetSetupSvc` was running.
- Reinstalling Tailscale, restarting Windows, stopping VMware virtual machines, and running `netsh int ip reset` did not resolve the issue.
- Testing from a separate local administrator account did not resolve it either.
- `%windir%\INF\setupapi.dev.log` showed repeated Wintun device cleanup attempts containing:

  ```text
  Query-and-Remove failed: 0x05: CR_INVALID_DEVNODE
  ```

- This registry key existed, and its default value was `VMware Bridge notifier object`:

  ```text
  HKLM\SOFTWARE\Classes\CLSID\{3d09c1ca-2bcc-40b7-b9bb-3f3ec143a87b}
  ```

## Workaround that worked on this machine

**Caution:** Removing this key may affect VMware bridged networking. Close or shut down running virtual machines first. Back up the exact key before changing it, and avoid deleting any other VMware or network-driver registry entries.

1. Open PowerShell as Administrator.
2. Export a backup of the exact key:

   ```powershell
   New-Item -ItemType Directory -Path 'D:\Tailscale-Diag' -Force | Out-Null
   reg.exe export 'HKLM\SOFTWARE\Classes\CLSID\{3d09c1ca-2bcc-40b7-b9bb-3f3ec143a87b}' 'D:\Tailscale-Diag\VMware-Bridge-CLSID-backup.reg' /y
   ```

   Continue only if the export says the operation completed successfully and the file exists.

3. Delete only that CLSID key:

   ```powershell
   reg.exe delete 'HKLM\SOFTWARE\Classes\CLSID\{3d09c1ca-2bcc-40b7-b9bb-3f3ec143a87b}' /f
   ```

4. Restart Windows.
5. Verify the network adapter status:

   ```powershell
   Get-CimInstance Win32_PnPEntity |
       Where-Object { $_.PNPDeviceID -like 'SWD\WINTUN*' } |
       Select-Object Name, ConfigManagerErrorCode
   ```

   The result on the affected machine changed from Code `56` to `0`.

6. Start/authenticate Tailscale and verify the connection:

   ```powershell
   tailscale up
   tailscale status
   ```

   In this case, `tailscale up` completed successfully after browser authentication and `tailscale status` displayed the tailnet device list.

## Rollback

If VMware bridged networking is affected, restore the saved registry key from an elevated PowerShell session:

```powershell
reg.exe import 'D:\Tailscale-Diag\VMware-Bridge-CLSID-backup.reg'
```

Restart Windows afterward and test VMware networking.

## What this result does and does not prove

- **Observed:** On this computer, deleting the backed-up VMware Bridge notifier CLSID and restarting Windows changed the Tailscale adapter status from Code 56 to Code 0, after which Tailscale authenticated successfully.
- **Not proven:** That VMware is the root cause in every Code 56 case, or that deleting this key is safe or appropriate on every VMware installation.
- Try non-destructive diagnostics first and create a backup or restore point before registry changes.
- Before posting logs publicly, redact hostnames, tailnet IPs, email addresses, authentication URLs, and other identifying information.

## Related reports

- [Tailscale issue #21406: Windows Wintun adapter installation failure](https://github.com/tailscale/tailscale/issues/21406) — related Windows/Wintun device-creation symptoms; the reported problem code differs.
- [Tailscale conflicts with VMware — TheCyberSimon](https://writeups.thecybersimon.com/posts/tailscale-conflicts-with-vmware/) — a separate report describing a VMware Bridge notifier CLSID and Tailscale Code 56.
