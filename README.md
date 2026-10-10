# Cybersecurity Lab Notes

Personal learning journal for red team fundamentals: Linux, networking, and reconnaissance.

## Contents

| Directory | What's inside |
|---|---|
| [`lab-setup/`](lab-setup/) | KVM/QEMU lab architecture, VM setup, network design |
| [`bandit/`](bandit/) | OverTheWire Bandit progress log (passwords redacted) |
| [`wireshark/`](wireshark/) | Protocol analysis notes: ICMP, TCP, DNS |
| [`screenshots/`](screenshots/) | Curated screenshots referenced from other docs |

## Lab Architecture
Host (Arch Linux + Hyprland)
└── KVM/QEMU + libvirt
└── virbr-lab (isolated bridge, 10.66.66.0/24)
├── kali (attacker) 10.66.66.X
└── metasploitable2 (target) 10.66.66.Y

Both VMs run on an isolated network with no route to the physical LAN or the internet.
This is a deliberate design — a compromised target must never reach real infrastructure.

## What I'm Learning

- **Networking:** TCP/IP fundamentals, subnetting, packet analysis with Wireshark
- **Linux:** CLI fluency via OverTheWire Bandit, filesystem navigation, `find`/`grep`/`awk`
- **Virtualization:** KVM/QEMU/libvirt, virtual networks, virtual disk management
- **Coming next:** Service enumeration, exploitation, privilege escalation

## Notable Troubleshooting

Real issues encountered while building the lab:

- **`sch_htb` kernel module missing** — libvirt's default network could not start because the HTB qdisc module wasn't available on the running kernel. Fixed by installing the mainline kernel and loading the module persistently.
- **`libvirt` group activation** — group membership changes do not apply to running sessions; a re-login is required before `virsh` works unprivileged.
- **libvirt URI confusion** — `virsh` defaults to `qemu:///session`, which shows no VMs. Set `LIBVIRT_DEFAULT_URI="qemu:///system"` to fix.
- **Metasploitable2 boot failure** — the 2.6.24 kernel lacks virtio-blk support. Rebuilt the VM with `--machine pc` and `bus=ide`.
- **Wireshark filter mistake** — using `||` (OR) instead of `&&` (AND) matches far more than intended.

## Rules I Follow

- Bandit passwords are **never** committed. Notes reference the technique, not the answer.
- Packet captures and VM images are excluded via `.gitignore`.
- No real IPs, hostnames, or credentials from any production environment.

## References

- [OverTheWire Bandit](https://overthewire.org/wargames/bandit/)
- [libvirt documentation](https://libvirt.org/docs.html)
- [Wireshark User's Guide](https://www.wireshark.org/docs/wsug_html_chunked/)
