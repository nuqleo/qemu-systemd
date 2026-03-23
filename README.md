This repository provides a simple and structured way to run QEMU virtual machines using systemd units, with networking handled by systemd-networkd and nftables.

It is intended for lightweight environments where virtual machines are managed through plain configuration files without additional virtualization layers.

---

## Features

- systemd template units for running QEMU virtual machines
- Support for multiple architectures:
  - x86_64
  - ARM
  - RISC-V
- Per-VM configuration files
- Bridge networking via systemd-networkd
- Built-in IPv4 DHCP server
- IPv6 SLAAC support via Router Advertisements
- Static DHCP leases support
- NAT using nftables

---

## Directory Structure

```

qemu-systemd
├── etc
│   ├── nftables
│   │   └── qemu-nat.nft
│   ├── qemu
│   │   ├── test-arm.conf
│   │   ├── test-riscv.conf
│   │   └── test-vm.conf
│   └── systemd
│       ├── network
│       │   ├── virbr0.netdev
│       │   └── virbr0.network
│       └── system
│           ├── [qemu-arm@.service](mailto:qemu-arm@.service)
│           ├── [qemu-riscv@.service](mailto:qemu-riscv@.service)
│           └── [qemu@.service](mailto:qemu@.service)

````

---

## Requirements

- QEMU
- systemd
- systemd-networkd
- nftables

---

## Installation

Copy files into system directories:

```bash
cp -r etc/systemd/system/* /etc/systemd/system/
cp -r etc/systemd/network/* /etc/systemd/network/
cp -r etc/qemu /etc/
cp etc/nftables/qemu-nat.nft /etc/nftables/
````

Reload systemd:

```bash
systemctl daemon-reexec
systemctl daemon-reload
```

Enable and start networking:

```bash
systemctl enable systemd-networkd
systemctl restart systemd-networkd
```

Apply nftables rules:

```bash
nft -f /etc/nftables/qemu-nat.nft
```

---

## Usage

### Start a virtual machine

x86_64:

```bash
systemctl start qemu@test-vm
```

ARM:

```bash
systemctl start qemu-arm@test-arm
```

RISC-V:

```bash
systemctl start qemu-riscv@test-riscv
```

### Enable autostart

```bash
systemctl enable qemu@test-vm
```

---

## VM Configuration

VM configuration files are stored in:

```
/etc/qemu/
```

Each file contains QEMU command-line arguments.

Example:

```bash
-name test-vm
-m 1024
-smp 2
-drive file=disk.img,format=qcow2
-netdev bridge,id=hn0,br=virbr0
-device virtio-net-pci,netdev=hn0,mac=52:54:00:12:34:56
```

---

## Networking

The provided systemd-networkd configuration:

* Creates a bridge interface `virbr0`
* Provides IPv4 and IPv6 connectivity for virtual machines

### Bridge usage

Virtual machine interfaces are attached to the bridge via QEMU configuration:

```bash
-netdev bridge,id=hn0,br=virbr0
-device virtio-net-pci,netdev=hn0
```

---

### IPv4

* `virbr0` runs an internal DHCP server via systemd-networkd
* Virtual machines receive IPv4 addresses automatically

Static leases are configured in `virbr0.network`:

```ini
[DHCPServerStaticLease]
MACAddress=52:54:00:12:34:56
Address=192.168.100.10
```

---

### IPv6

* Router Advertisements are enabled on the bridge
* Virtual machines configure IPv6 addresses automatically (SLAAC)

Configuration example from `virbr0.network`:

```ini
[Network]
IPv6SendRA=yes

[IPv6Prefix]
Prefix=2001:db8::/64
```

* `[IPv6Prefix]` defines the subnet advertised to virtual machines

---

### NAT

* Outbound connectivity is provided via nftables
* Rules are defined in:

```
/etc/nftables/qemu-nat.nft
```

* IPv4 and IPv6 NAT can be configured depending on host setup

---

## Notes

* No VM lifecycle management beyond systemd
* No built-in storage management
* Disk images must be managed manually
* Designed for simple and reproducible setups

---
```
