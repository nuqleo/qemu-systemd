This repository provides a simple and structured way to run QEMU virtual machines using systemd units, with networking handled by systemd-networkd and nftables.

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
- `nc` (netcat, e.g. netcat-openbsd or nmap-ncat) — used by systemd unit to send `system_powerdown` to VM

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
- `QEMU_ARGS` — QEMU command-line arguments

The systemd service uses `QEMU_ARCH` to select the appropriate QEMU binary:
```
/usr/bin/qemu-system-${QEMU_ARCH}
```
Make sure the corresponding binary is installed on the system.

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
-machine type=q35,accel=kvm,usb=on \
-device virtio-scsi-pci,id=scsi0,num_queues=4 \
-drive file=/path/to/vm.qcow2,format=qcow2,if=none,id=hd0,cache=unsafe,discard=unmap,detect-zeroes=unmap \
-device scsi-hd,bus=scsi0.0,drive=hd0 \
-netdev bridge,br=virbr0,id=net0 \
-device virtio-net-pci,netdev=net0,speed=10000,mac=00:01:02:03:04:05 \
-device virtio-tablet \
-device virtio-vga \
-display vnc=127.0.0.1:10 \
-serial telnet:127.0.0.1:2010,server,nowait \
-bios /usr/share/edk2/ovmf/OVMF_CODE.fd \
-boot menu=off"
```
The systemd service reads the `QEMU_ARGS` variable and passes it directly to the QEMU binary.

## Service User

QEMU is executed under a dedicated unprivileged user.

* On Fedora, this user is typically qemu
* On other systems, the user name may differ depending on the distribution

You may need to adjust the user configuration in the systemd unit files depending on your system.

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
This socket provides access to the QEMU monitor (HMP - Human Monitor Protocol) and can be used to control the virtual machine.

Example session:

```text
$ nc -U /run/qemu-<instance>.sock
(qemu) sendkey ctrl-alt-delete
```
You can then interactively send QEMU monitor commands.

---

## Networking

The provided systemd-networkd configuration:

* Creates a bridge interface `virbr0`
* Provides IPv4 and IPv6 connectivity for virtual machines

### Bridge usage

Virtual machine interfaces are attached to the bridge via QEMU configuration:

```bash
-netdev bridge,br=virbr0,id=net0
-device virtio-net-pci,netdev=net0,speed=10000,mac=00:01:02:03:04:05
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

### VFIO and memory locking

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
