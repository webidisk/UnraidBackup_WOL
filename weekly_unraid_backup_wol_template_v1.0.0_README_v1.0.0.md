# Weekly Unraid-to-Unraid Backup — Wake-on-LAN Template

**Documentation revision:** v1.0.0  
**Backup script:** `weekly_unraid_backup_wol_template_v1.0.0.sh`  
**Report helper:** `weekly_unraid_report_wol_template_v1.0.0.sh`  
**Branch:** Wake-on-LAN template, based on the feature set in the IPMI/BMC backup script `v1.2.0`

---

## 1. Purpose

This package is for an Unraid source server that backs up to a second Unraid server which **does not have BMC/IPMI**, but can be powered on through Wake-on-LAN (WoL).

The workflow is:

1. A weekly User Scripts job starts on the **source** Unraid server.
2. The source sends a Wake-on-LAN magic packet to the powered-off backup server.
3. The script waits for the backup server to answer ping, accept SSH, and mount `/mnt/user`.
4. On a real run, Docker is disabled on the backup server.
5. On a real run, Docker is temporarily disabled on the source server while `appdata` and `system` are backed up.
6. Source Docker is re-enabled and all remaining shares continue copying.
7. A report is generated with transferred files, new versus updated files, bytes copied, and storage size information.
8. The backup server shuts down cleanly through SSH with `shutdown -h now`.

This is intentionally **no-delete** backup behavior. Files removed from the source server are not automatically deleted from the backup server.

---

## 2. Files in This Package

| File | Purpose | Installation location |
|---|---|---|
| `weekly_unraid_backup_wol_template_v1.0.0.sh` | Main scheduled backup script | Paste into User Scripts on the source server. |
| `weekly_unraid_report_wol_template_v1.0.0.sh` | End-of-run report generator | `/boot/config/backup_scripts/weekly_unraid_report_wol_template_v1.0.0.sh` on the source server. |
| `weekly_unraid_backup_wol_template_v1.0.0_README_v1.0.0.md` | Installation and operations guide | Keep with the script release. |

All commands below are run on the **source server** unless the step says **backup server**.

---

## 3. Values You Must Replace

Open `weekly_unraid_backup_wol_template_v1.0.0.sh` and replace all placeholders before running it:

| Placeholder | What to enter |
|---|---|
| `REPLACE_WITH_SOURCE_SERVER_IP` | IP address of the source/main Unraid server. |
| `REPLACE_WITH_BACKUP_SERVER_IP` | IP address of the destination/backup Unraid server. |
| `REPLACE_WITH_BACKUP_SERVER_MAC` | MAC address of the backup server NIC that supports WoL. Use `aa:bb:cc:dd:ee:ff` format. |
| `REPLACE_WITH_NETWORK_BROADCAST_IP` | Broadcast address of the backup server LAN/VLAN. |
| `REPLACE_WITH_SOURCE_SERVER_NETWORK_INTERFACE` | Source interface such as `br0` or a VLAN bridge; only required when using `WOL_METHOD="etherwake"`. |

Find any remaining placeholders before running:

```bash
grep -n 'REPLACE_WITH_' /boot/config/plugins/user.scripts/scripts/REPLACE_WITH_USER_SCRIPT_NAME/script
```

The script refuses to run until required placeholders are replaced.

---

## 4. Backup Policy and Exclusions

The script copies source shares to matching paths on the backup server:

```text
Source: /mnt/user/SHARE_NAME/
Target: /mnt/user/SHARE_NAME/
```

Behavior:

| Scenario | Result |
|---|---|
| File exists only on source | Copied to backup. |
| File exists on both and source changed | Backup current copy is updated. |
| File exists only on backup | Retained; it is not deleted. |
| File was accidentally deleted on source | Existing backup-only copy remains available. |

Current exclusions:

| Excluded path | Reason |
|---|---|
| `/mnt/user/CCTVStorage` | CCTV recordings are excluded. |
| `/mnt/user/BackupReports` | Prevents the source report/log share from backing up itself. |
| `/mnt/user/system/docker/docker.img` | Large Docker image; typically recreated from templates and appdata. |
| `/mnt/user/system/libvirt/libvirt.img` | Large libvirt image; excluded by this template. |
| Common `appdata` cache/temp/log/runtime files | Reduces nonessential churn and transient rsync warnings. |

---

## 5. Prerequisites

Confirm the following:

- User Scripts is installed on the source Unraid server.
- Both servers have a wired Ethernet connection.
- The backup server motherboard and active NIC support Wake-on-LAN from shutdown.
- Wake-on-LAN is enabled in the backup server BIOS/UEFI.
- If BIOS has an ErP/EuP option, it is disabled when required for WoL.
- You can log in as `root` on both Unraid servers during setup.
- The backup server's array or pool provides `/mnt/user` after it boots.

Unraid's documentation notes that WoL requires NIC support, BIOS/UEFI configuration, Ethernet networking, and testing on the actual hardware; manual `ethtool` WoL settings are not persistent unless added to `/boot/config/go`.

---

## 6. Configure Wake-on-LAN on the Backup Server

### Step 6.1 — Enable WoL in BIOS/UEFI

On the **backup server**, open BIOS/UEFI setup and enable a setting such as:

```text
Wake on LAN
PME Event Wake-up
Power On by PCI/PCIe Device
Wake from S5
```

If present and required by the motherboard, disable:

```text
ErP / EuP Ready
```

Save settings and boot Unraid.

### Step 6.2 — Identify the network interface and MAC address

Run on the **backup server**:

```bash
ip -br link
ip -br addr
```

Locate the physical NIC that stays connected while the server is powered off. Record its MAC address and use it for:

```bash
DEST_MAC="REPLACE_WITH_BACKUP_SERVER_MAC"
```

### Step 6.3 — Verify and enable WoL in Unraid

Replace `REPLACE_WITH_BACKUP_SERVER_NETWORK_INTERFACE` with the active physical NIC name on the backup server, such as `eth0`:

```bash
ethtool REPLACE_WITH_BACKUP_SERVER_NETWORK_INTERFACE | grep -i 'Supports Wake-on\|Wake-on'
```

A suitable device usually reports:

```text
Supports Wake-on: ...g...
Wake-on: g
```

If `Wake-on` is disabled, enable it:

```bash
ethtool -s REPLACE_WITH_BACKUP_SERVER_NETWORK_INTERFACE wol g
```

### Step 6.4 — Make the WoL NIC setting persistent

Manual WoL interface settings can be lost at reboot. Add the enable command to the backup server startup file:

```bash
printf '\n/sbin/ethtool -s REPLACE_WITH_BACKUP_SERVER_NETWORK_INTERFACE wol g\n' >> /boot/config/go
```

Review the file to ensure the line was not added more than once:

```bash
tail -20 /boot/config/go
```

---

## 7. Configure Persistent SSH Transfer from Source to Backup Server

The backup data path uses SSH and rsync; Wake-on-LAN is used only for startup.

### Step 7.1 — Create the private/public key on the source server

```bash
mkdir -p /boot/config/ssh_keys
chmod 700 /boot/config/ssh_keys

ssh-keygen -t ed25519 \
  -f /boot/config/ssh_keys/id_ed25519_unraid_backup \
  -N ""

chmod 600 /boot/config/ssh_keys/id_ed25519_unraid_backup
chmod 644 /boot/config/ssh_keys/id_ed25519_unraid_backup.pub
```

Never copy the private key file off the source server.

### Step 7.2 — Install the public key on the backup server

Run from the source server. You will enter the backup server `root` password during initial setup:

```bash
cat /boot/config/ssh_keys/id_ed25519_unraid_backup.pub | \
ssh root@REPLACE_WITH_BACKUP_SERVER_IP '
  read -r NEW_KEY
  mkdir -p /boot/config/ssh/root
  touch /boot/config/ssh/root/authorized_keys
  if ! grep -qxF "$NEW_KEY" /boot/config/ssh/root/authorized_keys; then
    printf "%s\n" "$NEW_KEY" >> /boot/config/ssh/root/authorized_keys
  fi
  chmod 700 /boot/config/ssh/root
  chmod 600 /boot/config/ssh/root/authorized_keys
'
```

### Step 7.3 — Create the dedicated known-hosts file on the source server

```bash
mkdir -p /boot/config/ssh_keys
ssh-keyscan -H REPLACE_WITH_BACKUP_SERVER_IP > /boot/config/ssh_keys/known_hosts_unraid_backup
chmod 600 /boot/config/ssh_keys/known_hosts_unraid_backup
```

### Step 7.4 — Test passwordless SSH access

Make sure the backup server is powered on for this test:

```bash
ssh -i /boot/config/ssh_keys/id_ed25519_unraid_backup \
  -o UserKnownHostsFile=/boot/config/ssh_keys/known_hosts_unraid_backup \
  -o StrictHostKeyChecking=yes \
  -o UpdateHostKeys=no \
  -o BatchMode=yes \
  root@REPLACE_WITH_BACKUP_SERVER_IP \
  'hostname && date && mountpoint -q /mnt/user && echo /mnt/user-is-mounted'
```

Do not continue until this command connects without requesting a password and prints `/mnt/user-is-mounted`.

---

## 8. Install the Reporting Helper

On the source server:

```bash
mkdir -p /boot/config/backup_scripts
```

Copy `weekly_unraid_report_wol_template_v1.0.0.sh` into:

```text
/boot/config/backup_scripts/weekly_unraid_report_wol_template_v1.0.0.sh
```

Then run:

```bash
chmod 700 /boot/config/backup_scripts/weekly_unraid_report_wol_template_v1.0.0.sh
bash -n /boot/config/backup_scripts/weekly_unraid_report_wol_template_v1.0.0.sh
```

---

## 9. Install the Main Backup Script in User Scripts

On the source Unraid server:

1. Open **Settings → User Scripts**.
2. Add a new script, for example `Weekly Backup to Standby Unraid - WOL`.
3. Paste the contents of `weekly_unraid_backup_wol_template_v1.0.0.sh`.
4. Replace the required placeholder values.
5. Leave this setting in safe test mode initially:

```bash
DRY_RUN="yes"
```

Check Bash syntax from the source terminal after saving the User Script:

```bash
bash -n /boot/config/plugins/user.scripts/scripts/REPLACE_WITH_USER_SCRIPT_NAME/script
```

---

## 10. Wake-on-LAN Method Used by the Template

Default configuration:

```bash
WOL_METHOD="bash_udp"
```

This sends a standard WoL magic packet from Bash using the configured broadcast address and does not require a separate WoL package on the source server.

Alternative methods are available:

```bash
WOL_METHOD="etherwake"
WOL_METHOD="wakeonlan"
```

When using `etherwake`, also replace:

```bash
WOL_INTERFACE="REPLACE_WITH_SOURCE_SERVER_NETWORK_INTERFACE"
```

When using `etherwake` or `wakeonlan`, that command must already be installed on the source server.

---

## 11. First Startup Test from a Powered-Off Backup Server

1. Confirm `DRY_RUN="yes"` in the script.
2. Shut down the backup server cleanly.
3. Run the User Script manually.
4. Follow its timestamped log:

```bash
tail -f "$(ls -1t /mnt/user/BackupReports/WeeklyUnraidBackupWOL/logs/weekly_unraid_backup_wol_*.log | head -n 1)"
```

Expected startup messages include:

```text
Backup server appears offline. Sending Wake-on-LAN magic packet.
Wake-on-LAN packets sent. Waiting ... seconds before ping checks.
Backup server responds to ping.
Backup server SSH is ready and /mnt/user is mounted.
DRY RUN enabled. MAIN Docker will not be disabled for critical backup.
```

A dry run intentionally does not shut down the backup server at the end. Shut it down after the test with:

```bash
ssh -i /boot/config/ssh_keys/id_ed25519_unraid_backup \
  -o UserKnownHostsFile=/boot/config/ssh_keys/known_hosts_unraid_backup \
  -o StrictHostKeyChecking=yes \
  -o UpdateHostKeys=no \
  root@REPLACE_WITH_BACKUP_SERVER_IP 'sync && shutdown -h now'
```

---

## 12. First Real End-to-End Test

After the dry-run startup test succeeds, change:

```bash
DRY_RUN="no"
```

Run once manually. The real workflow should:

```text
Wake destination with WoL
→ wait for /mnt/user
→ disable destination Docker
→ disable source Docker for appdata/system only
→ back up appdata and system
→ restore source Docker
→ back up remaining shares, excluding CCTVStorage and BackupReports
→ generate report
→ shut down destination cleanly
```

---

## 13. Weekly Schedule

In User Scripts, use a custom cron schedule. Example: once per week on Sunday at 2:00 AM:

```cron
0 2 * * 0
```

Schedule the job only after the manual end-to-end test succeeds.

---

## 14. Logs and Reports

The script stores verbose logs on source array storage instead of the boot flash device:

```text
/mnt/user/BackupReports/WeeklyUnraidBackupWOL/logs/
```

Reports are stored at:

```text
/mnt/user/BackupReports/WeeklyUnraidBackupWOL/reports/
```

To tail the newest log:

```bash
tail -f "$(ls -1t /mnt/user/BackupReports/WeeklyUnraidBackupWOL/logs/weekly_unraid_backup_wol_*.log | head -n 1)"
```

The final report includes:

- Total files transferred.
- Files newly created versus existing files updated.
- Folders created.
- Bytes transferred by share and overall.
- Current source and destination share sizes, when enabled.
- Source and destination filesystem capacity information.

---

## 15. Troubleshooting

### Backup server does not wake

Confirm:

- The NIC link light remains active while the backup server is off.
- BIOS/UEFI WoL settings are enabled.
- ErP/EuP is disabled if it prevents wake from shutdown.
- `ethtool REPLACE_WITH_BACKUP_SERVER_NETWORK_INTERFACE` reports `Wake-on: g` before shutdown.
- `DEST_MAC` is the MAC of the wired NIC connected to the LAN.
- `WOL_BROADCAST` is the correct broadcast address for the destination network.

### Backup server responds to ping but backup does not start

The script waits for both SSH and `/mnt/user` to be mounted. A slow array/pool start is normal; increase `SSH_WAIT_ATTEMPTS` if necessary.

### Existing destination files

Files already present on the backup server are checked by rsync. Matching files are skipped; source files that have changed update the destination; destination-only files remain because deletion is disabled.

### Rsync returns code 24

Code 24 indicates files vanished during transfer. The script records it as a warning and continues because transient appdata files can disappear during a scan.

### Stale PID lock after interrupted run

Check for an active transfer before removing the lock directory:

```bash
pgrep -af 'rsync .*REPLACE_WITH_BACKUP_SERVER_IP:/mnt/user/'
```

When no old transfer is active:

```bash
rm -rf /var/run/weekly_unraid_backup_wol.lockdir
```

### Check main Docker status after interruption

```bash
grep '^DOCKER_ENABLED=' /boot/config/docker.cfg
/etc/rc.d/rc.docker status
docker ps
```

---

## 16. Version History

| Version | Change |
|---|---|
| `v1.0.0-wol-template` | Initial Wake-on-LAN template fork: placeholder addresses, no BMC/IPMI dependency, Bash UDP magic packet sender, no-delete backup policy, Docker critical-pass handling, report generation, clean shutdown, PID lock protection. |

---

## 17. Reference

Unraid official documentation: Wake-on-LAN (WoL), including requirements, BIOS/UEFI guidance, `ethtool` enablement, persistence through `/boot/config/go`, and testing considerations.
