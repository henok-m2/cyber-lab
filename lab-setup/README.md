# Lab Setup

How the isolated KVM lab is built and how to bring it up after a reboot.

## Host

- Arch Linux, kernel 6.x mainline (not LTS — LTS lacks `sch_htb`)
- Hyprland (Wayland)
- KVM/QEMU + libvirt (legacy `libvirtd`, not modular stack)
- Storage: `/var/lib/libvirt/images/` on NVMe

## Kernel Module Requirement

libvirt attaches an HTB traffic-shaping qdisc to every virtual bridge it creates.
This requires the `sch_htb` kernel module. On some kernel builds it isn't loaded
by default, causing `virsh net-start default` to fail with:

```
error: internal error: Child process (tc qdisc add dev virbr0 root handle 1: htb default 2)
unexpected exit status 2: Error: Specified qdisc kind is unknown.
```

Fix:

```bash
sudo modprobe sch_htb
echo "sch_htb" | sudo tee /etc/modules-load.d/libvirt.conf
```

The `modules-load.d` file makes the module load at boot, before libvirtd starts.

## User Permissions

Add self to the `libvirt` group:

```bash
sudo usermod -aG libvirt $USER
```

**Log out and back in** — group changes don't apply to running sessions.

Set the default libvirt URI in your shell rc (`~/.zshrc` for zsh):

```bash
export LIBVIRT_DEFAULT_URI="qemu:///system"
```

Without this, `virsh` defaults to the per-user `qemu:///session` and shows no VMs.

## Networks

### `default` — NAT network (unused)

Standard libvirt network. Provides internet access to VMs. Not used for the lab
because it would route lab traffic out to the internet.

### `labnet` — isolated network (used)

```xml
<network>
  <name>labnet</name>
  <bridge name="virbr-lab" stp="on" delay="0"/>
  <ip address="10.66.66.1" netmask="255.255.255.0">
    <dhcp>
      <range start="10.66.66.100" end="10.66.66.200"/>
    </dhcp>
  </ip>
</network>
```

**Critical:** no `<forward>` element. This tells libvirt not to install any NAT or
routing rules for this bridge. VMs on `labnet` can reach each other and the host's
bridge IP (`10.66.66.1`) but nothing else.

Defined at `/etc/libvirt/qemu/networks/labnet.xml`.

```bash
sudo virsh net-define /etc/libvirt/qemu/networks/labnet.xml
sudo virsh net-start labnet
sudo virsh net-autostart labnet
```

## VMs

| Name | Purpose | Disk bus | NIC | Video | RAM | vCPU |
|---|---|---|---|---|---|---|
| `kali` | Attacker | `virtio` | `virtio` | `qxl` | 4 GiB | 2 |
| `metasploitable2` | Target | `ide` | `e1000` | `cirrus` | 1 GiB | 1 |

The device asymmetry is deliberate: Metasploitable2's kernel is 2.6.24 (2008) and
predates virtio. Using modern devices fails with `ALERT! /dev/mapper/metasploitable-root
does not exist`. Fix: `--machine pc` and `bus=ide`.

## Day-to-Day Operation

Start everything:

```bash
virsh start kali
virsh start metasploitable2
virt-viewer --connect qemu:///system kali &
virt-viewer --connect qemu:///system metasploitable2 &
```

Get DHCP leases:

```bash
virsh net-dhcp-leases labnet
```

Shut down cleanly:

```bash
virsh shutdown kali
virsh shutdown metasploitable2
```

## Isolation Verification

From inside either VM:

```bash
ping -c 2 10.66.66.1      # host bridge — should succeed
ping -c 2 192.168.1.1     # your LAN — should fail with "Network is unreachable"
ping -c 2 8.8.8.8         # internet — should fail with "Network is unreachable"
```

`Network is unreachable` means the kernel found no route — the strongest possible
isolation signal. Nothing leaves the VM.
