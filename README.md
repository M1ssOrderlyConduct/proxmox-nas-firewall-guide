# Proxmox VE 9 + pfSense + Suricata IPS + NAS Setup Guide

A comprehensive guide for setting up a secure home server with Proxmox VE 9, pfSense firewall with Suricata IPS/IDS, and NAS storage on an Odroid H4 Ultra (Intel i3-N305).

## Table of Contents

1. [Hardware Overview](#hardware-overview)
2. [Architecture Design](#architecture-design)
3. [Proxmox VE 9 Known Issues & Solutions](#proxmox-ve-9-known-issues--solutions)
4. [Pre-Installation Checklist](#pre-installation-checklist)
5. [Proxmox Installation](#proxmox-installation)
6. [IOMMU & Passthrough Configuration](#iommu--passthrough-configuration)
7. [pfSense VM Setup](#pfsense-vm-setup)
8. [Suricata IPS Configuration](#suricata-ips-configuration)
9. [NAS Configuration Options](#nas-configuration-options)
10. [Post-Installation Hardening](#post-installation-hardening)
11. [Troubleshooting](#troubleshooting)
12. [Maintenance & Monitoring](#maintenance--monitoring)

---

## Hardware Overview

### Target Hardware: Odroid H4 Ultra

| Component | Specification |
|-----------|---------------|
| CPU | Intel i3-N305 (8 cores, 8 threads, 3.8GHz) |
| RAM | 32GB DDR5 |
| NICs | 2x Intel i226-V 2.5GbE (ideal for pfSense WAN/LAN) |
| SATA | 4x SATA III ports |
| NVMe | 1x M.2 NVMe slot (boot drive) |
| Storage | 4x 14TB HDDs for NAS |

### Why This Hardware Works Well

- **Dual Intel NICs**: Native driver support in pfSense/OPNsense, perfect for WAN/LAN separation
- **8 Cores**: Adequate for Suricata IPS inline inspection + VM workloads
- **32GB RAM**: Enough for ZFS ARC cache + multiple VMs
- **Low Power**: ~15-35W TDP, suitable for 24/7 operation

---

## Architecture Design

```
                        ┌──────────────────────────────────────────────────────────────┐
                        │                      PROXMOX VE 9 HOST                        │
                        │                     (NVMe Boot Drive)                         │
                        ├──────────────────────────────────────────────────────────────┤
Internet ───┐           │                                                              │
            │           │  ┌────────────────────────┐  ┌────────────────────────────┐  │
            │           │  │     pfSense VM         │  │   TrueNAS Scale VM         │  │
            ▼           │  │   (4 vCPU, 4-8GB)      │  │   (4 vCPU, 16GB)           │  │
      ┌─────────┐       │  │                        │  │                            │  │
      │  WAN    │◄──────┼──┤  enp1s0 (passthrough)  │  │  SATA Controller or        │  │
      │  NIC    │       │  │                        │  │  4x 14TB Passthrough       │  │
      └─────────┘       │  │  ┌──────────────────┐  │  │                            │  │
                        │  │  │   SURICATA IPS   │  │  │  RAIDZ1 (~42TB usable)     │  │
      ┌─────────┐       │  │  │   Inline Mode    │  │  │  or RAIDZ2 (~28TB)         │  │
      │  LAN    │◄──────┼──┤  │   ET Open Rules  │  │  │                            │  │
      │  NIC    │       │  │  └──────────────────┘  │  │  SMB/NFS/iSCSI Shares      │  │
      └─────────┘       │  │                        │  │                            │  │
            │           │  │  enp2s0 (passthrough)  │  └────────────────────────────┘  │
            │           │  └────────────────────────┘                                  │
            ▼           │                                                              │
      Local Network     └──────────────────────────────────────────────────────────────┘
```

### Resource Allocation Strategy

| VM/Service | vCPUs | RAM | Storage | Priority |
|------------|-------|-----|---------|----------|
| Proxmox Host | 2 reserved | 4GB | NVMe | - |
| pfSense + Suricata | 4 | 4-8GB | 32GB NVMe | HIGH |
| TrueNAS Scale | 4 | 16GB | 16GB boot + 4x14TB | MEDIUM |
| Future VMs | Remaining | Remaining | As needed | LOW |

---

## Proxmox VE 9 Known Issues & Solutions

### Critical Issues

#### 1. Kernel 6.17.4 Boot Failures
**Symptom**: System won't boot after upgrading to kernel 6.17.4-1-pve
**Affected**: Dell systems, some Intel platforms
**Solution**: Pin kernel to 6.17.2-2-pve

```bash
# Boot into recovery or older kernel first
apt-mark hold proxmox-kernel-6.17.2-2-pve
apt-mark hold pve-kernel-6.17.2-2-pve

# Remove problematic kernel
apt remove proxmox-kernel-6.17.4-1-pve
update-grub
reboot
```

#### 2. Intel N100/N305 iGPU Passthrough Regression
**Symptom**: iGPU passthrough works in PVE 8.x but fails in PVE 9.x
**Status**: Known regression, awaiting fix
**Workaround**:
- For NAS/firewall use case, iGPU passthrough is NOT needed
- If needed for transcoding, stay on PVE 8.x or use LXC with bind mount

#### 3. IOMMU Feature Inconsistent Warnings
**Symptom**: dmesg shows multiple "IOMMU feature inconsistent" warnings
**Impact**: Usually cosmetic, passthrough still works
**Solution**: These can typically be ignored if passthrough functions correctly

```bash
# Check IOMMU groups are properly separated
find /sys/kernel/iommu_groups/ -type l | sort -V
```

### Upgrade Issues (PVE 8 to 9)

#### Missing pve-apt-hook Error
**Symptom**: `/usr/share/proxmox-ve/pve-apt-hook: not found` (error 127)
**Solution**:

```bash
mkdir -p /usr/share/proxmox-ve/
touch /usr/share/proxmox-ve/pve-apt-hook
chmod +x /usr/share/proxmox-ve/pve-apt-hook
```

#### Repository Configuration
**Important**: Update ALL repository files, not just sources.list.d

```bash
# /etc/apt/sources.list must contain:
deb http://deb.debian.org/debian trixie main contrib non-free non-free-firmware
deb http://deb.debian.org/debian trixie-updates main contrib non-free non-free-firmware
deb http://security.debian.org/debian-security trixie-security main contrib non-free non-free-firmware
```

#### Always Use dist-upgrade
```bash
# CORRECT - resolves dependencies properly
apt dist-upgrade

# WRONG - may leave packages on old versions
apt upgrade
```

### Deprecated Features in PVE 9

| Feature | Status | Alternative |
|---------|--------|-------------|
| cgroupv1 | Removed | cgroupv2 only (CentOS 7, Ubuntu 16.04 containers won't work) |
| GlusterFS | Dropped | Use Ceph, NFS, or iSCSI |
| Older NVIDIA GRID | Incompatible | Use GRID 18.3+ (driver 570.158.02+) |

### BIOS-Related Issues

**Symptom**: Boot failures after BIOS update
**Cause**: BIOS defaults may reset SATA mode to RAID
**Solution**:
1. Enter BIOS setup
2. Find Storage/SATA configuration
3. Change from "RAID" to "AHCI"
4. Save and reboot

---

## Pre-Installation Checklist

### BIOS Configuration

- [ ] **Intel VT-d**: ENABLED (required for PCI passthrough)
- [ ] **Intel VT-x**: ENABLED (required for virtualization)
- [ ] **SATA Mode**: AHCI (not RAID)
- [ ] **Secure Boot**: DISABLED (or configure with Proxmox keys)
- [ ] **CSM/Legacy Boot**: DISABLED (use UEFI only)
- [ ] **Wake on LAN**: Optional, enable if desired

### Hardware Preparation

- [ ] NVMe installed in M.2 slot (boot drive)
- [ ] 4x 14TB drives connected to SATA ports
- [ ] Both NICs connected (WAN to modem, LAN to switch)
- [ ] USB drive prepared with Proxmox ISO

### Network Information Needed

| Parameter | Value | Notes |
|-----------|-------|-------|
| WAN IP | DHCP or Static | From ISP |
| LAN Subnet | e.g., 192.168.1.0/24 | Your choice |
| pfSense LAN IP | e.g., 192.168.1.1 | Gateway for clients |
| Proxmox Mgmt IP | e.g., 192.168.1.2 | Static recommended |

---

## Proxmox Installation

### Step 1: Create Bootable USB

```bash
# On Linux/macOS
dd if=proxmox-ve_9.1-1.iso of=/dev/sdX bs=4M status=progress
sync

# On Windows: Use Rufus or Etcher
```

### Step 2: Install Proxmox

1. Boot from USB
2. Select "Install Proxmox VE (Graphical)"
3. Accept EULA
4. Select NVMe drive as target (NOT the 14TB drives!)
5. Set country/timezone/keyboard
6. Set root password and email
7. **Network Configuration** (IMPORTANT):
   - Management Interface: Select one NIC temporarily
   - Hostname: `pve.local` or your choice
   - IP: Static IP on your current network
   - Gateway: Your current router
   - DNS: Your current DNS

### Step 3: Post-Install Updates

```bash
# SSH into Proxmox or use web shell

# Remove enterprise repo (if no subscription)
rm /etc/apt/sources.list.d/pve-enterprise.list

# Add no-subscription repo
echo "deb http://download.proxmox.com/debian/pve trixie pve-no-subscription" > /etc/apt/sources.list.d/pve-no-subscription.list

# Update system
apt update && apt dist-upgrade -y
reboot
```

### Step 4: Disable Subscription Nag (Optional)

```bash
# Edit the JavaScript file
sed -Ezi.bak "s/(function\(orig_cmd\) \{)/\1\n\torig_cmd();\n\treturn;/g" /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js

# Clear browser cache and refresh
systemctl restart pveproxy
```

---

## IOMMU & Passthrough Configuration

### Enable IOMMU

```bash
# Edit GRUB configuration
nano /etc/default/grub

# Find GRUB_CMDLINE_LINUX_DEFAULT and modify:
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"

# Update GRUB
update-grub

# Add VFIO modules
echo "vfio" >> /etc/modules
echo "vfio_iommu_type1" >> /etc/modules
echo "vfio_pci" >> /etc/modules
echo "vfio_virqfd" >> /etc/modules

# Update initramfs
update-initramfs -u -k all

reboot
```

### Verify IOMMU is Working

```bash
# Check IOMMU is enabled
dmesg | grep -i iommu
# Should see: "DMAR: IOMMU enabled"

# List IOMMU groups
find /sys/kernel/iommu_groups/ -type l | sort -V

# Check NIC IOMMU groups (need separate groups for passthrough)
lspci -nnk | grep -A3 Ethernet
```

### Blacklist Host Drivers for Passthrough NICs

```bash
# Identify NIC PCI IDs
lspci -nn | grep -i ethernet
# Example output: 01:00.0 Ethernet [0200]: Intel Corporation I226-V [8086:125c]
# Example output: 02:00.0 Ethernet [0200]: Intel Corporation I226-V [8086:125c]

# Create blacklist
echo "options vfio-pci ids=8086:125c" > /etc/modprobe.d/vfio.conf
echo "blacklist igc" >> /etc/modprobe.d/blacklist.conf

# Update initramfs
update-initramfs -u -k all
reboot
```

### Verify NICs are Bound to VFIO

```bash
lspci -nnk | grep -A3 Ethernet
# Should show: Kernel driver in use: vfio-pci
```

---

## pfSense VM Setup

### Download pfSense ISO

1. Go to https://www.pfsense.org/download/
2. Select: AMD64, ISO Installer
3. Upload to Proxmox: Datacenter > pve > local > ISO Images > Upload

### Create pfSense VM

**General Tab:**
- VM ID: 100
- Name: pfSense

**OS Tab:**
- ISO: pfSense ISO
- Type: Other

**System Tab:**
- Machine: q35
- BIOS: OVMF (UEFI)
- Add EFI Disk: Yes (local-lvm)
- Pre-Enroll keys: Unchecked

**Disks Tab:**
- Bus: VirtIO Block
- Size: 32GB
- SSD emulation: Yes
- Discard: Yes

**CPU Tab:**
- Cores: 4
- Type: host

**Memory Tab:**
- Memory: 4096-8192 MB
- Ballooning: DISABLED (important!)

**Network Tab:**
- Delete the default network (we'll add passthrough NICs)

### Add Passthrough NICs

```bash
# In Proxmox shell, edit VM config
nano /etc/pve/qemu-server/100.conf

# Add these lines for PCI passthrough:
hostpci0: 01:00.0,pcie=1
hostpci1: 02:00.0,pcie=1
```

Or via Web UI:
1. Hardware > Add > PCI Device
2. Select first Intel NIC (Raw Device)
3. Check "PCI-Express"
4. Repeat for second NIC

### Install pfSense

1. Start VM, open Console
2. Accept defaults for most options
3. Install to virtual disk
4. Remove ISO and reboot
5. Assign interfaces:
   - WAN: First passthrough NIC (igc0 or similar)
   - LAN: Second passthrough NIC (igc1 or similar)
6. Set LAN IP (e.g., 192.168.1.1/24)
7. Access web GUI from LAN at https://192.168.1.1

### Critical pfSense Settings

```
System > Advanced > Networking:
- [ ] Disable hardware checksum offloading (CHECK THIS)
- [ ] Disable hardware TCP segmentation offloading (CHECK THIS)
- [ ] Disable hardware large receive offloading (CHECK THIS)

# These MUST be disabled for Suricata inline mode!
```

---

## Suricata IPS Configuration

### Install Suricata Package

1. System > Package Manager > Available Packages
2. Search "suricata"
3. Click Install

### Configure Global Settings

**Services > Suricata > Global Settings:**

| Setting | Value | Notes |
|---------|-------|-------|
| Enable Suricata VRT | Optional | Requires free registration |
| Enable ET Open | YES | Free, high-quality rules |
| Enable ET Pro | Optional | Paid, more rules |
| Hide Deprecated Rules | YES | Cleaner interface |
| Update Interval | 12 hours | Balance freshness vs bandwidth |

Click "Update" to download rules.

### Configure WAN Interface

**Services > Suricata > Interfaces > Add:**

| Setting | Value | Notes |
|---------|-------|-------|
| Interface | WAN | Primary protection point |
| Enable | YES | - |
| Block Offenders | YES | For IPS mode |
| IPS Mode | Inline Mode | TRUE IPS (not Legacy) |
| Block on DROP only | YES | Recommended - granular control |

### Rule Categories to Enable

**WAN Categories Tab** - Enable these for home/small business:

```
Essential (Always Enable):
- emerging-exploit
- emerging-malware
- emerging-trojan
- emerging-worm
- emerging-dos
- emerging-scan
- emerging-shellcode
- emerging-attack_response

Recommended:
- emerging-policy (blocks known bad)
- emerging-compromised (known bad IPs)
- emerging-botcc (botnet C&C)
- emerging-ciarmy (CI Army bad IPs)
- emerging-drop (Spamhaus DROP list)
- emerging-dshield (DShield bad IPs)

Optional (More Alerts):
- emerging-web_client
- emerging-web_server (if hosting)
- emerging-sql (if hosting databases)
```

### Performance Tuning

**Interface Settings > WAN:**

| Setting | Value | Notes |
|---------|-------|-------|
| Max Pending Packets | 1024 | Default is fine |
| Detect Engine Profile | Medium | Balance speed/accuracy |
| Pattern Matcher | Hyperscan | Fastest, requires SSE4.2 |
| Stream Memory Cap | 64MB | Increase if memory available |

### Suppress False Positives

Common false positives to suppress:

```
Services > Suricata > Suppress

# Google/Cloudflare DNS over HTTPS
suppress gen_id 1, sig_id 2027865

# Windows Update traffic
suppress gen_id 1, sig_id 2027758

# Common streaming services
suppress gen_id 1, sig_id 2027863
```

### Verify Suricata is Running

```bash
# In pfSense shell (Diagnostics > Command Prompt)
ps aux | grep suricata
# Should show suricata process with -i netmap:...

# Check for blocks
cat /var/log/suricata/suricata_em0*/eve.json | grep '"event_type":"alert"' | tail -20
```

---

## NAS Configuration Options

### Option A: TrueNAS Scale VM (Recommended)

**Pros:**
- Full ZFS feature set with GUI
- Automated snapshots, scrubs, SMART tests
- Apps ecosystem (Plex, Nextcloud, etc.)
- Excellent SMB/NFS support

**Cons:**
- Higher resource usage (16GB RAM minimum)
- Requires disk/controller passthrough

#### TrueNAS VM Setup

**Create VM:**
- VM ID: 101
- Name: TrueNAS
- OS: TrueNAS Scale ISO
- System: q35, OVMF (UEFI)
- Disk: 16GB VirtIO (boot only)
- CPU: 4 cores, type=host
- RAM: 16384 MB, Ballooning=OFF

**Passthrough Storage (Choose One):**

**Method 1: SATA Controller Passthrough (Best)**
```bash
# Find SATA controller
lspci | grep -i sata
# Example: 00:17.0 SATA controller: Intel Corporation...

# Add to VM config
echo "hostpci2: 00:17.0,pcie=1" >> /etc/pve/qemu-server/101.conf
```

**Method 2: Individual Disk Passthrough**
```bash
# Find disk IDs (stable across reboots)
ls -la /dev/disk/by-id/ | grep -v part

# Add each disk to VM config
qm set 101 -scsi1 /dev/disk/by-id/ata-WDC_WD140EFGX-68B0GN0_XXXXXXXX
qm set 101 -scsi2 /dev/disk/by-id/ata-WDC_WD140EFGX-68B0GN0_YYYYYYYY
qm set 101 -scsi3 /dev/disk/by-id/ata-WDC_WD140EFGX-68B0GN0_ZZZZZZZZ
qm set 101 -scsi4 /dev/disk/by-id/ata-WDC_WD140EFGX-68B0GN0_WWWWWWWW

# Set serial numbers (required by TrueNAS)
# Edit /etc/pve/qemu-server/101.conf and add serial= to each disk line
```

**TrueNAS Pool Configuration:**

| RAID Level | Usable Space | Fault Tolerance | Recommended For |
|------------|--------------|-----------------|-----------------|
| RAIDZ1 | ~42TB | 1 drive | Budget, less critical |
| RAIDZ2 | ~28TB | 2 drives | Recommended for NAS |
| Mirror (2 pairs) | ~28TB | 1 per pair | Best performance |

### Option B: Proxmox Native ZFS + LXC (Lightweight)

**Pros:**
- Lower overhead (no VM)
- Direct ZFS access
- Simpler setup

**Cons:**
- Manual snapshot configuration
- Less GUI polish
- DIY monitoring

#### Setup Steps

```bash
# Create ZFS pool (RAIDZ1 example)
zpool create -o ashift=12 tank raidz1 \
  /dev/disk/by-id/ata-WDC_WD140EFGX-68B0GN0_XXXXXXXX \
  /dev/disk/by-id/ata-WDC_WD140EFGX-68B0GN0_YYYYYYYY \
  /dev/disk/by-id/ata-WDC_WD140EFGX-68B0GN0_ZZZZZZZZ \
  /dev/disk/by-id/ata-WDC_WD140EFGX-68B0GN0_WWWWWWWW

# Create datasets
zfs create tank/media
zfs create tank/backups
zfs create tank/documents

# Set compression
zfs set compression=lz4 tank

# Enable NFS
apt install nfs-kernel-server
echo "/tank/media 192.168.1.0/24(rw,sync,no_subtree_check)" >> /etc/exports
exportfs -ra

# Enable SMB via Cockpit LXC or direct install
apt install samba
```

### Storage Recommendations

| Use Case | RAID Level | Settings |
|----------|------------|----------|
| Media Server | RAIDZ1 | recordsize=1M, compression=lz4 |
| Documents | RAIDZ2 | recordsize=128K, compression=lz4 |
| VMs/Databases | Mirror | recordsize=64K, sync=standard |
| Backups | RAIDZ1 | recordsize=1M, compression=zstd |

---

## Post-Installation Hardening

### Proxmox Host

```bash
# 1. Enable firewall
pvesh set /cluster/firewall/options --enable 1

# 2. Create firewall rules (Web UI: Datacenter > Firewall)
# Allow: SSH (22), Web GUI (8006) from LAN only

# 3. Disable root SSH password login
sed -i 's/^PermitRootLogin yes/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config
systemctl restart sshd

# 4. Create non-root admin user
pveum user add admin@pam
pveum aclmod / -user admin@pam -role Administrator

# 5. Enable 2FA (Web UI: Datacenter > Permissions > Two Factor)

# 6. Configure email alerts
apt install libsasl2-modules
# Edit /etc/postfix/main.cf for your SMTP relay
```

### pfSense Hardening

```
1. System > Advanced > Admin Access:
   - Enable HTTPS only
   - Disable webConfigurator redirect
   - Set session timeout (e.g., 240 minutes)

2. System > Advanced > Networking:
   - Disable all hardware offloading (for Suricata)

3. Firewall > Rules > WAN:
   - Block all by default (should already be)
   - Add specific allow rules as needed

4. Services > DNS Resolver:
   - Enable DNSSEC
   - Consider DoH/DoT upstream

5. System > User Manager:
   - Create non-admin user for daily use
   - Enable TOTP 2FA
```

### TrueNAS Hardening

```
1. System Settings > Services:
   - Disable SSH unless needed
   - Use key-based auth if enabled

2. Shares > SMB:
   - Use separate users per share
   - Enable SMB encryption if possible

3. Data Protection:
   - Enable automatic scrubs (weekly)
   - Enable automatic SMART tests (weekly short, monthly long)
   - Configure automated snapshots

4. Alerts:
   - Configure email alerts for disk failures
```

---

## Troubleshooting

### Suricata Not Blocking

**Symptom**: Alerts generated but traffic not blocked

**Checklist**:
1. Verify "IPS Mode" is set to "Inline Mode" (not Legacy)
2. Verify "Block Offenders" is enabled
3. Verify "Block on DROP only" and rules are set to DROP (not ALERT)
4. Check hardware offloading is DISABLED
5. Restart Suricata service

```bash
# Force restart
/usr/local/etc/rc.d/suricata.sh restart
```

### PCI Passthrough Not Working

**Symptom**: VM won't start, error about IOMMU or passthrough

**Checklist**:
1. Verify VT-d is enabled in BIOS
2. Verify IOMMU is enabled in kernel: `dmesg | grep -i iommu`
3. Check IOMMU groups: `find /sys/kernel/iommu_groups/ -type l`
4. Verify VFIO driver bound: `lspci -nnk | grep -A3 <device>`
5. Try adding `pcie_acs_override=downstream,multifunction` to GRUB (last resort)

### TrueNAS Can't See Disks

**Symptom**: Passed-through disks not appearing in TrueNAS

**Checklist**:
1. Verify disks are passed through (VM Hardware tab shows them)
2. Check serial numbers are set (TrueNAS requires this)
3. For controller passthrough, ensure AHCI mode in BIOS
4. Check TrueNAS console: `camcontrol devlist`

### Network Issues After pfSense Setup

**Symptom**: No internet access through pfSense

**Checklist**:
1. Verify WAN interface has IP (Status > Interfaces)
2. Check gateway is assigned (System > Routing)
3. Verify DNS is working: `ping 8.8.8.8` vs `ping google.com`
4. Check firewall rules (Firewall > Rules > WAN)
5. Check NAT (Firewall > NAT > Outbound)

---

## Maintenance & Monitoring

### Weekly Tasks

- [ ] Check Suricata alerts (Services > Suricata > Alerts)
- [ ] Review pfSense logs (Status > System Logs)
- [ ] Verify backups completed
- [ ] Check disk health (TrueNAS or smartctl)

### Monthly Tasks

- [ ] Update Proxmox: `apt update && apt dist-upgrade`
- [ ] Update pfSense: System > Update
- [ ] Update TrueNAS: System Settings > Update
- [ ] Review firewall rules
- [ ] Test restore from backup

### Monitoring Recommendations

| Tool | Purpose | Location |
|------|---------|----------|
| Proxmox Dashboard | Host metrics | Web UI |
| pfSense Status | Network/Suricata | pfSense Web UI |
| TrueNAS Alerts | Disk/Pool health | TrueNAS Web UI |
| Uptime Kuma (optional) | Service monitoring | Docker/LXC |
| Grafana + InfluxDB (optional) | Metrics visualization | Docker/LXC |

### Backup Strategy

```
Critical Backups:
1. Proxmox VM configs: /etc/pve/qemu-server/
2. pfSense config: Diagnostics > Backup & Restore
3. TrueNAS config: System Settings > General > Manage Configuration
4. TrueNAS datasets: Automated replication or cloud sync

Backup Destinations:
- Local: Separate ZFS dataset
- Offsite: Cloud storage (B2, Wasabi, etc.)
- 3-2-1 Rule: 3 copies, 2 media types, 1 offsite
```

---

## Quick Reference Commands

### Proxmox

```bash
# VM Management
qm list                          # List VMs
qm start 100                     # Start VM 100
qm stop 100                      # Stop VM 100
qm config 100                    # Show VM config

# Storage
pvesm status                     # Show storage status
zpool status                     # ZFS pool status
zfs list                         # List datasets

# Networking
cat /etc/network/interfaces      # Network config
ip a                             # Show interfaces
```

### pfSense Shell

```bash
# Suricata
/usr/local/etc/rc.d/suricata.sh status
/usr/local/etc/rc.d/suricata.sh restart

# Logs
clog /var/log/filter.log         # Firewall log
cat /var/log/suricata/*/eve.json # Suricata events

# Network
ifconfig                         # Interface status
netstat -rn                      # Routing table
pfctl -sr                        # Firewall rules
```

### TrueNAS Shell

```bash
# Pool/Dataset
zpool status                     # Pool health
zfs list -t snapshot             # List snapshots
zfs get all tank                 # All pool properties

# Services
midclt call service.query        # List services
midclt call service.restart "cifs"  # Restart SMB
```

---

## Sources & References

- [Proxmox VE 9.1 Release Notes](https://forum.proxmox.com/threads/proxmox-virtual-environment-9-1-available.176255/)
- [Proxmox 8 to 9 Upgrade Guide](https://pve.proxmox.com/wiki/Upgrade_from_8_to_9)
- [Common PVE 8-9 Upgrade Issues](https://www.saturnme.com/common-issues-during-proxmox-ve-8-to-9-upgrade-and-how-to-fix-them/)
- [Proxmox PCI Passthrough Wiki](https://pve.proxmox.com/wiki/PCI_Passthrough)
- [Intel N100/N305 GPU Passthrough Issues](https://forum.proxmox.com/threads/intel-n100-miniserver-gpu-passthrough-results-in-black-screen.132450/)
- [Proxmox 9 GPU Passthrough](https://forum.proxmox.com/threads/proxmox-9-gpu-pass-through.169151/)
- [pfSense Suricata Inline IPS Mode](https://forum.netgate.com/topic/96482/suricata-true-inline-ips-mode-coming-with-pfsense-2-3-here-is-a-preview)
- [Suricata Inline vs Legacy Mode](https://wiki.sharewiz.net/doku.php?id=pfsense:suricata:inline_versus_legacy_ips_mode)
- [OPNsense IPS Documentation](https://docs.opnsense.org/manual/ips.html)
- [TrueNAS on Proxmox Best Practices](https://forum.proxmox.com/threads/best-practice-for-truenas-virtualized-on-proxmox.133536/)
- [TrueNAS SATA Passthrough](https://forum.proxmox.com/threads/passing-an-entire-sata-hdd-to-truenas-scale-vm.153741/)
- [ZFS on Proxmox vs TrueNAS](https://discourse.practicalzfs.com/t/truenas-vs-zfs-in-proxmox/2139)

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-01-03 | Initial release |

---

*Guide created for Odroid H4 Ultra deployment. Adjust hardware-specific settings for other platforms.*
