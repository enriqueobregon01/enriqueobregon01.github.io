# Lab 01: Network Foundation & Segmentation

> **Summary:** Built a firewall-gated virtual network from scratch — a pfSense gateway isolating an Ubuntu Server host on a private LAN — designed the IP scheme by hand, and verified connectivity end to end from host through firewall to the internet, including DNS resolution.

| | |
|---|---|
| **Duration** | ~4 hrs (including troubleshooting) |
| **Environment** | Oracle VirtualBox 7.2 on a 16GB Windows host |
| **VMs** | pfSense CE 2.8.1-RELEASE (gateway), Ubuntu Server 26.04 LTS (host) |
| **Skills demonstrated** | Network design, static IP addressing, subnetting/CIDR/VLSM, firewall gateway setup, DNS configuration, layer-2/layer-3 troubleshooting |
| **Framework refs** | NIST SP 800-41 (firewall guidance); network segmentation as a defense-in-depth control |

---

## 1. Objective

Stand up the foundation the rest of the lab series builds on: a firewall-gated internal network with a host behind it. The goal was to demonstrate deliberate network design — a hand-planned IP scheme rather than DHCP-and-hope — and to prove every layer works: the host reaches its gateway, routes to the internet through the firewall, and resolves names via DNS. Segmentation is a foundational security control, so establishing it correctly here sets up the detection and defense labs that follow.

---

## 2. Topology

A single isolated lab network with no bridge to the physical home LAN (deliberate — later labs run scans and exploits that must stay contained). pfSense sits between a NAT-based "WAN" (for internet access and updates) and a host-only "LAN" where lab hosts live.

```
                 +-------------+
   Internet ---> |  pfSense    |  WAN: em0 -> NAT (DHCP: 10.0.2.15)
                 |  gateway    |  LAN: em1 -> 10.10.10.1/24
                 +------+------+
                        | host-only network
                        | "VirtualBox Host-Only Ethernet Adapter" (10.10.10.0/24)
                 +------+-------+
                 | Ubuntu Server|  enp0s3 -> 10.10.10.20/24
                 |  (lab host)  |  gateway 10.10.10.1
                 +--------------+

   Windows host also sits on the host-only network at 10.10.10.2,
   used to reach the pfSense web GUI at https://10.10.10.1
```

### IP scheme

| Host | Role | IP | Interface | Notes |
|---|---|---|---|---|
| pfSense (WAN) | Internet uplink | 10.0.2.15/24 (DHCP) | em0 -> NAT | Auto from VirtualBox NAT |
| pfSense (LAN) | Gateway/firewall | 10.10.10.1/24 | em1 -> host-only | The lab gateway |
| Ubuntu Server | Linux host / future target | 10.10.10.20/24 | enp0s3 -> host-only | Static via Netplan |
| Windows host | Management / GUI access | 10.10.10.2/24 | host-only adapter | Reaches pfSense GUI |
| (reserved) | Kali (Lab 03) | 10.10.10.40/24 | host-only | Gaps left intentionally |

Addresses were spaced (`.20`, `.30`, `.40`) to keep the scheme readable and leave room — a habit that mirrors real network documentation.

> **[Insert: draw.io topology diagram — export to `evidence/topology.png`]**

---

## 3. Build Steps

### 3.1 VirtualBox host-only network
Created a host-only network (`10.10.10.0/24`) with **DHCP disabled** — addressing is assigned by hand and by pfSense, not by VirtualBox. The host adapter holds `10.10.10.2`; pfSense owns the gateway `.1`.

### 3.2 pfSense gateway VM
- Two adapters: **Adapter 1 = NAT** (becomes WAN), **Adapter 2 = Host-only** (becomes LAN).
- Installed via the Netgate net-installer (see Troubleshooting #2).
- Assigned interfaces at the console: **WAN = em0**, **LAN = em1**, matching adapter MACs.
- Set LAN to `10.10.10.1/24` during install.
- Filesystem: ZFS (stripe, single disk), GPT partition scheme.

### 3.3 Ubuntu Server host — static IP via Netplan
`/etc/netplan/50-cloud-init.yaml`:
```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: false
      addresses:
        - 10.10.10.20/24
      routes:
        - to: default
          via: 10.10.10.1
      nameservers:
        addresses:
          - 10.10.10.1
```
```bash
sudo netplan apply
```

### 3.4 pfSense DNS
Ran the setup wizard, set upstream DNS to `8.8.8.8` / `1.1.1.1`, enabled the DNS Resolver in **forwarding mode** with DNSSEC disabled (see Troubleshooting #6).

---

## 4. Verification

From the Ubuntu host:

```bash
ip addr show enp0s3      # -> 10.10.10.20/24 bound to enp0s3
ip route                 # -> default via 10.10.10.1
ping -c 4 10.10.10.1     # gateway -> 0% loss
ping -c 4 8.8.8.8        # internet -> 0% loss, ttl 62 (decremented, proving routing through pfSense)
ping -c 4 google.com     # DNS -> resolves and replies
```

The TTL dropping from 64 to 62 on the `8.8.8.8` replies is direct evidence the traffic is routing *through* the pfSense gateway (each hop decrements TTL), not taking some shortcut.

> **[Insert: `evidence/ubuntu-ip.png`, `evidence/ping-success.png`, `evidence/pfsense-console.png`]**

---

## 5. Troubleshooting

This lab surfaced a realistic string of failures. Each is documented with symptom -> diagnosis -> fix.

**1. pfSense VM would not boot — "No bootable option or device was found."**
Symptom: UEFI/BdsDxe errors on first boot. Diagnosis: the VM was attempting UEFI boot and the optical drive was not first in the boot order. Fix: in System -> Motherboard, disabled EFI (legacy BIOS boot) and moved Optical to the top of the boot order. The net-installer is more reliable under legacy BIOS in VirtualBox.

**2. Unexpected installer filename — `netgate-installer-v1.2` instead of `pfSense-CE-2.7.2.iso`.**
Symptom: the downloaded file did not match the classic pfSense CE ISO name. Diagnosis: verified that Netgate now distributes pfSense CE via a net-installer that downloads the pfSense packages over the internet at install time, rather than as a single large ISO. Fix: confirmed it was legitimate, kept the NAT/WAN adapter so the installer could reach Netgate's servers, and chose "Install CE" (not Plus) when prompted.

**3. Installer reboot loop.**
Symptom: after install completed and the VM rebooted, it booted straight back into the installer. Diagnosis: the installation ISO was still mounted in the virtual optical drive, so the VM booted from CD again. Fix: ejected the ISO via Devices -> Optical Drives -> Remove disk, then reset — the VM then booted from disk into pfSense.

**4. Ubuntu auto-install failed — "Username is reserved by the system: admin."**
Symptom: the Ubuntu VM dropped to an error shell during install. Diagnosis: VirtualBox's unattended-installation feature had triggered and auto-filled the reserved username `admin`, which Ubuntu rejects. Fix: recreated the VM and left the ISO unattached during the New VM wizard (attaching it afterward via Settings -> Storage), which prevented VirtualBox from launching unattended install and let the manual installer run.

**5. No connectivity — "Destination Host Unreachable" from the host itself.**
Symptom: pings to the gateway failed with the host reporting unreachable from its own address; the host had also pulled a `10.0.2.15` address. Diagnosis: `ip link` showed a single interface, and the `10.0.2.15` address is a VirtualBox NAT-range signature — impossible on a host-only network. The Ubuntu VM's Adapter 1 had been left on **NAT** instead of host-only, so it was on the wrong virtual switch and could not reach pfSense at layer 2. Fix: powered off the VM, switched Adapter 1 to Host-only, rebooted. Pings to the gateway and internet then succeeded. (Also confirmed the golden rule: the pfSense VM must be running for anything to route — it *is* the gateway.)

**6. DNS timed out despite correct settings — the Save-vs-Apply catch.**
Symptom: routing worked (`ping 8.8.8.8` succeeded) but name resolution failed; `dig @10.10.10.1` and even `host google.com 127.0.0.1` *on pfSense itself* timed out. Diagnosis: worked methodically — confirmed unbound was listening on `*:53` (`sockstat -4 -l | grep :53`), confirmed LAN firewall rules allowed the traffic, then found that local resolution failed too, isolating the problem to unbound's configuration rather than the client or network. Root cause: DNS Resolver settings had been changed in the GUI but never **applied** — pfSense requires both **Save** *and* clicking the **Apply Changes** banner for changes to go live. Fix: re-entered the settings (forwarding mode on, DNSSEC off, upstream DNS `8.8.8.8`/`1.1.1.1`), saved, and clicked Apply. Resolution worked immediately.

---

## 6. What I Learned

- A default route sends all off-subnet traffic to the gateway; without pfSense running, a host on the LAN can reach nothing beyond its own segment.
- The VirtualBox NAT range (`10.0.2.x`) is a reliable fingerprint — seeing it on a supposedly host-only interface immediately flags a misattached adapter.
- TTL decrement is concrete, verifiable proof that traffic is passing through a router/firewall hop.
- "Service is listening" (`sockstat` shows `*:53`) does not mean "service is working" — unbound was bound correctly yet still failing, because its config was never applied.
- In pfSense, configuration is a two-step commit: **Save** writes the form, **Apply Changes** makes it live. Skipping the second step produces settings that look correct but have no effect — a subtle failure mode worth internalizing.

---

## 7. Security Relevance

Network segmentation is one of the highest-leverage defensive controls an analyst works with: it limits lateral movement, contains a breach to a blast radius, and creates chokepoints where traffic can be inspected. By placing a firewall between the host and everything else on day one, this lab establishes the exact structure the later labs use to detect and contain a simulated intrusion — the same defense-in-depth pattern used in enterprise networks. The troubleshooting also reinforced a security-relevant habit: verifying that a control is not just *configured* but *actually in effect*, since a rule that was saved-but-not-applied is functionally the same as no rule at all.
