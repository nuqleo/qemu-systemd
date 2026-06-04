This repository provides a simple and structured way to run [QEMU](https://gitlab.com/qemu-project/qemu) virtual machines using systemd units, with networking handled by systemd-networkd and nftables.

It is intended for lightweight environments where virtual machines are managed through plain configuration files without additional virtualization layers.

---

## Features

- systemd template units for running QEMU virtual machines
- Support for multiple architectures:
  - x86_64
  - AArch64
  - LoongArch64
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
└─ etc
   ├── nftables
   │   └── qemu-nat.nft
   ├── qemu
   │   ├── test-arm.conf
   │   ├── test-loong.conf
   │   ├── test-riscv.conf
   │   └── test-x86.conf
   └── systemd
       ├── network
       │   ├── virbr0.netdev
       │   └── virbr0.network
       └── system
           └── qemu@.service

````

---

## Requirements

- QEMU
- systemd
- systemd-networkd
- nftables
- `nc` (netcat, e.g. OpenBSD netcat or nmap-ncat) — used by systemd unit to send `system_powerdown` to VM

---

## Installation

Copy files into system directories:

```bash
cp etc/systemd/system/qemu@.service /etc/systemd/system/
cp etc/systemd/network/virbr0.* /etc/systemd/network/
cp etc/nftables/qemu-nat.nft /etc/nftables/
cp -r etc/qemu /etc/
````

Reload systemd configuration:

```bash
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
systemctl start qemu@test-x86
```

AArch64:

```bash
systemctl start qemu@test-arm
```

LoongArch64:

```bash
systemctl start qemu@test-loong
```

RISC-V:

```bash
systemctl start qemu@test-riscv
```

### Enable autostart

```bash
systemctl enable qemu@test-x86
```

---

## VM Configuration

VM configuration files are stored in:

```
/etc/qemu/
```
Each configuration file defines:

- `QEMU_ARCH` — target architecture
- `QEMU_ARGS` — [QEMU command-line](https://www.qemu.org/docs/master/system/invocation.html) arguments

The systemd service uses `QEMU_ARCH` to select the appropriate QEMU binary:
```
/usr/bin/qemu-system-${QEMU_ARCH}
```
The value of `QEMU_ARCH` must match an available QEMU system emulator installed on the host.

Examples include:

- `x86_64`
- `aarch64`
- `riscv64`
- `loongarch64`

Additional architectures may be used if supported by the installed QEMU packages.

### Environment file syntax

Configuration files in `/etc/qemu/` are systemd environment files, not shell scripts.

When using line continuations (`\`), make sure:

- the backslash is the **last character on the line**
- there are **no trailing spaces or characters after `\`**
- the value is properly quoted

Example:

```bash
QEMU_ARCH="x86_64"

QEMU_ARGS="-m 8G \
-smp 16,sockets=2,cores=8,threads=1 \
-cpu host,kvm=off \
-machine type=q35,accel=kvm,firmware=/usr/share/edk2/ovmf/OVMF_CODE.fd \
-device virtio-scsi,num_queues=4 \
-drive file=/path/to/vm.qcow2,format=qcow2,if=none,id=hd0,cache=unsafe,discard=unmap,detect-zeroes=unmap \
-device scsi-hd,drive=hd0 \
-netdev bridge,br=virbr0,id=net0 \
-device virtio-net,netdev=net0,speed=10000,duplex=full,mac=00:01:02:03:04:05 \
-device virtio-tablet \
-device virtio-vga \
-display vnc=127.0.0.1:10 \
-serial telnet:127.0.0.1:2010,server,nowait \
-boot menu=off"
```
The systemd service reads the `QEMU_ARGS` variable and passes it directly to the QEMU binary.

## Service User

QEMU is executed under a dedicated unprivileged user.

* On Fedora, this user is typically `qemu`
* On other systems, the user name may differ depending on the distribution

You may need to adjust the user configuration in the systemd unit file depending on your system.

### Service stop behavior

The systemd service sends a `system_powerdown` signal to the VM when stopping.

- By default, the service waits **60 seconds** for the VM to shut down gracefully.
- This timeout can be adjusted via the `TimeoutStopSec` option in the `[Service]` section of the unit file:

```ini
TimeoutStopSec=90
```
### Service startup order

The QEMU systemd service depends on the bridge interface (e.g. `virbr0`) being available.

The unit is configured with:

```ini
Wants=sys-subsystem-net-devices-virbr0.device
After=sys-subsystem-net-devices-virbr0.device
```
This ensures that the network interface exists before the VM starts.

Make sure that systemd-networkd is running and has already configured the bridge interface before starting QEMU services.
If the interface is not present, QEMU may fail to start with bridge-related errors.

### QEMU monitor socket

Each VM exposes a QEMU monitor over a UNIX socket.

The socket path is defined in the systemd unit, for example:

```bash
/run/qemu-<instance>.sock
```
This socket provides access to the [QEMU monitor](https://www.qemu.org/docs/master/system/monitor.html) and can be used to control the virtual machine.

Example session:

```text
$ nc -U /run/qemu-<instance>.sock
(qemu) help
(qemu) sendkey ctrl-alt-delete
```
You can then interactively send QEMU monitor commands.

---

## Networking

The provided [systemd-networkd](https://www.freedesktop.org/software/systemd/man/latest/systemd.network.html) configuration:

* [Creates](https://www.freedesktop.org/software/systemd/man/latest/systemd.netdev.html) a bridge interface `virbr0`
* Provides IPv4 and IPv6 connectivity for virtual machines

### Bridge usage

Virtual machine interfaces are attached to the bridge via QEMU configuration:

```bash
-netdev bridge,br=virbr0,id=net0
-device virtio-net,netdev=net0,speed=10000,duplex=full,mac=00:01:02:03:04:05
```

### Multiple bridges

In addition to `virbr0`, you can define additional bridges (e.g. `virbr1`, `virbr2`, etc.) using the same systemd-networkd configuration approach.

To allow QEMU to use these bridges, they must be permitted in:

```bash
/etc/qemu/bridge.conf
```
Add entries like:
```ini
allow virbr1
allow virbr2
```
Or allow all bridges:
```ini
allow all
```
Without this configuration, QEMU will refuse to attach to the bridge.
This restriction is enforced by the QEMU bridge helper.

If you use additional bridges, you should also extend the systemd unit dependencies accordingly.

For example:
```ini
Wants=sys-subsystem-net-devices-virbr1.device
After=sys-subsystem-net-devices-virbr1.device
```
This ensures that all required bridge interfaces are available before the VM starts.

---

### IPv4

* `virbr0` runs an internal DHCP server via systemd-networkd
* Virtual machines receive IPv4 addresses automatically

Static leases are configured in `virbr0.network`:

```ini
[DHCPServerStaticLease]
MACAddress=00:01:02:03:04:05
Address=192.168.122.2
Hostname=test-x86
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

## MACVTAP Networking

As an alternative to bridge-based networking, virtual machines can be connected directly to the external Layer 2 network using MACVTAP interfaces instead of a software bridge.

In this setup, a dedicated MACVTAP interface is typically created for each virtual machine. The corresponding VM configuration file must define the `MACVTAP` variable:

```ini
MACVTAP="macvtap0"
```

The systemd service uses this interface to obtain the associated `/dev/tap*` device and passes it to QEMU.

### Interface configuration

A MACVTAP interface must be attached to a parent network interface. For example, to connect a VM to `eth0`:

```ini
[Match]
Name=eth0

[Network]
MACVTAP=macvtap0
```
Additional virtual machines can use their own MACVTAP interfaces (`macvtap1`, `macvtap2`, etc.) attached to the same or a different parent interface.

### MAC address consistency

The MAC address configured for the virtual machine must match the MAC address assigned to the corresponding MACVTAP interface.

Example:

```ini
# macvtap0.netdev
[NetDev]
Name=macvtap0
Kind=macvtap
MACAddress=00:01:02:03:04:05
```

```bash
# test-macvtap.conf
-device virtio-net,netdev=net0,mac=00:01:02:03:04:05
```

If the MAC addresses do not match, network connectivity will not work correctly.

### Service startup order

The `qemu-macvtap@.service` unit depends on the corresponding MACVTAP interface being available before the VM starts:

```ini
Wants=sys-subsystem-net-devices-macvtap0.device
After=sys-subsystem-net-devices-macvtap0.device
```

If a different interface name is used, the unit dependencies should be adjusted accordingly.

### DHCP and IPv6 configuration

Unlike the bridge-based configuration, systemd-networkd does not provide DHCP services or IPv6 Router Advertisements to virtual machines when using MACVTAP.

Virtual machines are connected directly to the external network and must obtain their network configuration from services available on that network, such as:

* External DHCP server
* External IPv6 Router Advertisements
* Static network configuration

### Host-to-VM communication

MACVTAP interfaces cannot communicate directly with the host through the parent interface.

If host-to-VM connectivity is required, an additional MACVLAN interface may be created on the same parent interface. In the provided example this is `macvlan0`:

```ini
[Network]
MACVLAN=macvlan0
```

The `macvlan0` interface is optional and is only needed when communication between the host and the virtual machine is required.

### Example files

```
qemu-systemd
└─ etc
   ├── qemu
   │   └── test-macvtap.conf
   └── systemd
       ├── network
       │   ├── eth0.network
       │   ├── macvlan0.netdev
       │   ├── macvlan0.network
       │   ├── macvtap0.netdev
       │   └── macvtap0.network
       └── system
           └── qemu-macvtap@.service
```
### Starting a VM

Virtual machines using MACVTAP networking are started with the `qemu-macvtap@.service` unit.

Example:

```bash
systemctl start qemu-macvtap@test-macvtap
```

Enable autostart:

```bash
systemctl enable qemu-macvtap@test-macvtap
```

## VFIO and memory locking

If you use VFIO (PCI passthrough), you may need to increase or remove memory locking limits.

Add the following to the `[Service]` section of the systemd unit:

```ini
LimitMEMLOCK=infinity
```

## Notes

* No VM lifecycle management beyond systemd
* No built-in storage management
* Disk images must be managed manually
* Designed for simple and reproducible setups

## License

This project is released into the public domain using CC0 1.0.
