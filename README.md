# SOC L1 Homelab — Splunk SIEM (VMware + Physical Endpoint)

A self-hosted Security Operations Center environment built to practice real SOC L1
workflows — log ingestion, field extraction, and (in progress) adversary simulation
and detection engineering — using a mix of virtual machines and a real physical
endpoint, so the telemetry reflects an actual device, not just isolated lab VMs.

> **Status: 🟡 In Progress.** This is a living project. Accurate as of July 2026.

---

## Architecture

```
┌─────────────────────────────────────────┐        ┌─────────────────────────┐
│         VMware Workstation (local)         │        │   Physical ThinkPad X230  │
│                                             │        │   (Arch Linux + i3wm,     │
│  ┌───────────────┐   ┌──────────────────┐ │        │    formerly Fedora 43)    │
│  │  Kali Linux    │   │  Ubuntu Server    │ │        │    connects over home     │
│  │  - attacker     │   │  Splunk Enterprise│◄┼────────┤    WiFi, forwards logs    │
│  │    endpoint      │   │  (SIEM indexer)    │ │  UF   │    via Universal          │
│  │  - also running   │   │                    │ │ 9997  │    Forwarder              │
│  │    scripted UF    │   │                    │ │       │                            │
│  │    input (journal) │   │                    │ │       │                            │
│  └───────────────┘   └──────────────────┘ │        └─────────────────────────┘
│                                             │
└─────────────────────────────────────────┘
       Home router (real DHCP, real LAN) — Ubuntu Server: 192.168.100.187
```

- **Hypervisor:** VMware Workstation (locally hosted)
- **SIEM:** Splunk Enterprise on a dedicated Ubuntu Server VM
- **Attacker endpoint:** Kali Linux VM
- **Real endpoint (not a VM):** the ThinkPad X230 itself, connected over home WiFi — chosen deliberately so the ingested telemetry reflects an actual physical device on a real network, not just VM-to-VM traffic on an isolated virtual switch
- **Networking:** Started fully inside VMware's internal network (`192.168.94.x`). Reconfigured to bridge onto the real home LAN (landing on `192.168.100.187` for the Ubuntu Server) specifically so the physical laptop — reachable only over WiFi — could deliver logs to the indexer

## What's Built So Far

- ✅ Deployed Splunk Enterprise on a VMware-hosted Ubuntu Server VM (after an earlier Proxmox attempt hit a hard resource ceiling — see [Challenges](#challenges--fixes))
- ✅ Installed Splunk Universal Forwarder on both the Kali VM and the physical ThinkPad endpoint
- ✅ Configured the indexer to receive forwarder traffic on TCP port 9997
- ✅ Forwarding real OS-level logs: `syslog`, `auth.log` (`linux_secure`), systemd journal data
- ✅ Wrote a **custom scripted input** (`stream_journal.sh`) on the Kali VM to stream its systemd journal into Splunk as `sourcetype=linux_systemd`, after discovering `splunk add script` isn't a valid UF subcommand and had to be configured differently (as a scripted input via `inputs.conf` instead)
- ✅ Verified end-to-end delivery — moved from `0.0.0.0:9997` inactive forwards, to confirmed active forwarding, to real events landing in Splunk (**106 events** over 24h from the `fedora` host alone during one verification pass)
- ✅ Worked raw → structured data using Splunk's automatic sourcetype extraction (`linux_secure`, `syslog`, `linux_systemd`) and manual extraction (Field Extractor, `| spath`) for less-standard payloads

## What's Next

- ⬜ Run adversary simulations from the Kali VM (Nmap reconnaissance, brute-force attempts) against the Ubuntu Server / physical endpoint
- ⬜ Capture that traffic in Wireshark to validate log fidelity against the actual attack signatures
- ⬜ Build SPL correlation searches to detect the simulated attacks
- ⬜ Convert working searches into real Splunk alerts (saved searches with trigger conditions)
- ⬜ Document each detection individually in [`/detections`](./detections) — query, MITRE ATT&CK mapping, false-positive notes

## Challenges & Fixes

**Problem:** Proxmox VE hit a hard resource ceiling — couldn't allocate compute
for an additional VM even after clearing temp data and expanding storage.
**Fix:** Abandoned Proxmox for this project and rebuilt on VMware Workstation
instead, where hardware allocation wasn't a blocker.

---

**Problem:** After expanding the Ubuntu Server's virtual disk in VMware, the OS
didn't automatically recognize the new space.
**Fix:** Manually extended the volume group and filesystem:
```bash
sudo growpart /dev/sda 3
sudo pvresize /dev/sda3
sudo lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv
sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
```

---

**Problem:** Universal Forwarders repeatedly fell into an inactive state
(`0.0.0.0:9997`) after the network was reconfigured (moving off VMware's
internal `192.168.94.x` range onto the real home LAN so the physical laptop
endpoint could reach the indexer). The stale `outputs.conf` kept pointing at
the old address.
**Fix:**
```bash
sudo /opt/splunkforwarder/bin/splunk stop
sudo rm /opt/splunkforwarder/etc/system/local/outputs.conf
sudo /opt/splunkforwarder/bin/splunk start
sudo /opt/splunkforwarder/bin/splunk add forward-server 192.168.100.187:9997
```

---

**Problem:** Wanted the Kali VM's own systemd journal streaming into Splunk,
but `sudo splunk add script ...` returned `Command error: The subcommand
'script' is not valid for command 'add'.`
**Fix:** Realized scripted inputs aren't added via that CLI subcommand — wrote
`stream_journal.sh` and registered it as a scripted input instead, confirmed
working with `linux_systemd`-sourcetyped events landing in Splunk from the
Kali host.

---

**Problem (hardware, not Splunk):** The ThinkPad X230 (16GB RAM) running
Fedora 43 (GNOME/Sway) suffered heavy lag — especially in Firefox — and rapid
overheating, despite the RAM being more than sufficient on paper.
**Fix:** Wiped Fedora, migrated the daily-driver/endpoint laptop to Arch Linux
with i3wm. Immediately lighter and cooler. Still actively tuning the i3 config
(polybar, picom, rofi) — this is an ongoing side effort, separate from the
Splunk lab itself.

## Verification Commands Reference

| Purpose | Command |
|---|---|
| List monitored log files on an endpoint | `sudo /opt/splunkforwarder/bin/splunk list monitor` |
| Check forwarder connection status | `sudo /opt/splunkforwarder/bin/splunk list forward-server` |
| Connect UF to indexer | `sudo /opt/splunkforwarder/bin/splunk add forward-server <indexer-ip>:9997` |
| Test raw connectivity to indexer port | `nc -zv <indexer-ip> 9997` |
| Base search validation | `index=main sourcetype=linux_secure` |

## Screenshots

*(To be added — see [`/screenshots`](./screenshots) for the capture checklist)*

## Repo Structure

```
soc-homelab/
├── README.md              ← you are here
├── screenshots/           ← infrastructure, ingestion, and search evidence
├── detections/             ← one write-up per detection rule (once built)
└── notes/
    └── build-log.md        ← full chronological build/troubleshooting log
```
