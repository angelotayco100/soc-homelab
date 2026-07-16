# Build Log — SOC Homelab

Chronological technical notes. More detailed/raw than the main README —
useful as personal reference and interview prep.

## Environment

- **Hypervisor:** VMware Workstation (locally hosted)
- **Splunk server:** Ubuntu Server VM, Splunk Enterprise
- **Attacker endpoint:** Kali Linux VM
- **Real endpoint:** ThinkPad X230, 16GB RAM — connected over home WiFi (not a VM).
  Originally Fedora 43, migrated to Arch Linux + i3wm after performance issues.

## Why a physical endpoint instead of another VM

Deliberate choice: monitoring only VM-to-VM traffic on an isolated virtual
switch doesn't reflect how a real endpoint behaves on an actual network. Using
the physical ThinkPad over home WiFi means the ingested telemetry comes from
a real device with a real DHCP lease on the real LAN, not simulated
VM-internal traffic.

## Hypervisor History: Proxmox → VMware

Started on Proxmox VE. Got as far as a running VM named `splunk-siem` (4GB
RAM, 2 cores, 12GB disk) before hitting a hard resource ceiling — Proxmox
couldn't allocate compute for an additional VM even after clearing temp data.
Abandoned Proxmox for this project and rebuilt on VMware Workstation instead.

## Networking History

1. **Phase 1 — VMware internal network:** VMs communicated on VMware's
   internal range (`192.168.94.x`). Kali attacker VM initially at
   `192.168.94.130`ish range, Ubuntu Server (Splunk) at `192.168.94.133`.
2. **Phase 2 — Bridged to home LAN:** Reconfigured networking so the Ubuntu
   Server pulls a real DHCP lease from the home router instead of staying on
   VMware's internal network. This was necessary so the physical ThinkPad
   (reachable only over WiFi) could actually reach the indexer. Ubuntu
   Server landed on `192.168.100.187`.

## Splunk Core Setup

- Installed Splunk Enterprise on the Ubuntu Server VM
- Installed Splunk Universal Forwarder at `/opt/splunkforwarder/` on:
  - The Kali Linux VM
  - The physical ThinkPad endpoint
- Configured the indexer to listen on port **9997** (hit `Failed to create.
  Configuration for port 9997 already exists` once — meant it was already
  configured, not a real blocker)
- Connected forwarders to the indexer:
  ```bash
  sudo /opt/splunkforwarder/bin/splunk add forward-server 192.168.100.187:9997
  ```
- Verified raw TCP reachability before troubleshooting Splunk-level config:
  ```bash
  nc -zv 192.168.94.133 9997
  # 192.168.94.133: inverse host lookup failed: Unknown host
  # (UNKNOWN) [192.168.94.133] 9997 (?) open
  ```

## Data Ingestion

Log sources successfully flowing into Splunk:
- `/var/log/syslog`
- `/var/log/auth.log` (sourcetype `linux_secure`)
- systemd journal data (via the custom scripted input — see below)

**Verifying what's being monitored locally:**
```bash
sudo /opt/splunkforwarder/bin/splunk list monitor
```

**Verifying the forwarder is actually connected to the indexer:**
```bash
sudo /opt/splunkforwarder/bin/splunk list forward-server
```
Success = moving from `0.0.0.0:9997` / inactive, to an active forward pointed
at the correct indexer IP.

**Confirmed working example search** (from the `fedora` host, before the
Arch migration):
```spl
index=main host="fedora"
| stats count by sourcetype, _raw
```
Returned 106 events over a 24h window, sourcetype `linux_systemd`.

## Custom Scripted Input — Kali Journal Streaming

Wanted the Kali VM's own systemd journal streaming into Splunk. First
attempt used the wrong CLI approach:
```bash
sudo /opt/splunkforwarder/bin/splunk add script /opt/splunkforwarder/bin/scripts/stream_journal.sh -interval 0 -sourcetype linux_systemd
# Command error: The subcommand 'script' is not valid for command 'add'.
```
`add script` isn't a real UF subcommand. Correct approach: wrote
`stream_journal.sh` directly and registered it as a **scripted input**
(via `inputs.conf`), not through `splunk add`. Confirmed the script was
executable and owned correctly:
```bash
ls -l /opt/splunkforwarder/bin/scripts/stream_journal.sh
# -rwxr-xr-x 1 splunkfwd splunkfwd 35 Jun 17 22:06 .../stream_journal.sh
```
Verified working — search `index=* host="kali-attacker"` returned 6 real
events sourced from `/opt/splunkforwarder/bin/scripts/stream_journal.sh`,
sourcetype `linux_systemd` (e.g. NetworkManager-dispatcher service events).

## Raw → Structured Data

- Standard logs (`syslog`, `linux_secure`) get baseline field extraction
  automatically via Splunk's built-in sourcetypes.
- For more complex payloads, used Splunk's **Field Extractor** and the
  **`| spath`** SPL command.
- Base validation searches used during this phase:
  ```spl
  index=main sourcetype=linux_secure
  ```

## Problems Encountered

### 1. Proxmox resource ceiling
**Cause:** Couldn't allocate resources for an additional VM even after
clearing temp data / expanding storage.
**Fix:** Abandoned Proxmox, rebuilt on VMware Workstation.

### 2. LVM didn't recognize expanded VMware disk
**Fix:**
```bash
sudo growpart /dev/sda 3
sudo pvresize /dev/sda3
sudo lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv
sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
```

### 3. Forwarders went inactive after network reconfiguration
**Symptom:** `splunk list forward-server` showed `0.0.0.0:9997`, inactive.
**Cause:** Stale `outputs.conf` still pointed at the old VMware-internal IP
after moving the Ubuntu Server onto the real home LAN.
**Fix:**
```bash
sudo /opt/splunkforwarder/bin/splunk stop
sudo rm /opt/splunkforwarder/etc/system/local/outputs.conf
sudo /opt/splunkforwarder/bin/splunk start
sudo /opt/splunkforwarder/bin/splunk add forward-server 192.168.100.187:9997
```

### 4. `splunk add script` isn't a valid subcommand
See "Custom Scripted Input" section above — resolved by using a proper
scripted input registration instead.

### 5. Package install friction (early on)
Multiple early hiccups getting Splunk installed via `.deb`:
- `wget -0` typo (should've been `-O`) — usage error
- `apt install ./splunk-*.deb` — "Unsupported file...given on commandline"
  (relative path issue)
- `dpkg -i` failing with "cannot access archive: No such file or directory"
  — wrong working directory / stale `cd` paths
- Eventually resolved by confirming exact file location with `ls -lh`,
  cleaning a corrupted partial download (`unexpected end of file in
  archive`), and re-running `dpkg -i` from the correct path with disk space
  confirmed via `df -h`.

### 6. (Hardware, not Splunk) Fedora 43 performance on the X230
**Symptom:** Heavy lag (especially Firefox), rapid overheating — despite
16GB RAM.
**Fix:** Wiped Fedora, migrated to Arch Linux + i3wm. Noticeably lighter,
faster, cooler. i3 config (polybar, picom, rofi, kitty) still being actively
tuned.

## Explicitly Not Done Yet

- ❌ No Nmap scanning has been run from the Kali VM against target endpoints
- ❌ No packet capture/analysis has been done in Wireshark
- ❌ No Splunk alerts (saved searches with trigger conditions) have been built

These are the next milestones — see README "What's Next."
