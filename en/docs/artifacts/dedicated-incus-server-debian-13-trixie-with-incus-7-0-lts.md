# dedicated Incus server Debian 13 (trixie) with Incus 7.0 LTS



For a dedicated Incus server I'd use **Debian 13 (trixie) with Incus 7.0 LTS**. It also slots straight into the Ventoy baseline you've just built.

**Why Debian 13:**

- **Kernel:** Incus 7.0 raised its minimum requirements: Minimum supported versions now include Linux kernel 6.12, Go 1.25, QEMU 8.2, LXC 6.0, nftables 1.0, and dnsmasq 2.90. Debian 13 ships kernel 6.12, so it meets that with its stock kernel. Ubuntu 24.04's original 6.8 kernel does not; you'd need the HWE kernel.
- **Packaging:** Debian packages LTS releases of Incus, beginning with trixie. Backported versions of new major Incus LTS releases are also available for bookworm and trixie. Alternatively, the upstream Zabbly repository offers multiple release channels including LTS (6.0, 7.0), Stable, and Daily. On a server, pin the `lts-7.0` channel.
- **Support length:** Incus 7.0 LTS will be supported until June 2031. That's a long, predictable runway for client infrastructure.
- **Upstream's own choice:** Incus's own appliance OS is built on top of Debian 13 "Trixie", so Debian is the base the Incus developers themselves test against.
- **Fits your plan:** it's free, has no vendor tie-in, and is already covered by your Ventoy preseed and Ansible handover.

**Storage: move off the loop file.** On your Mint laptop a file-backed Btrfs pool is fine. On a dedicated server, give Incus a real partition or a whole disk. Install `btrfs-progs`, then create the pool on the block device. You keep everything you like about Btrfs, minus the extra filesystem-on-a-file layer. Btrfs is a well-supported backend: Incus 7.4 added near-live container migration on local ZFS or Btrfs storage, which helps if you later run more than one host. ZFS is the other strong option, and it's what upstream favours.

**IncusOS, the other candidate, and why I wouldn't pick it here.** It's the purpose-built option: an immutable operating system based on Debian 13, purpose-built to host the Incus server. But it doesn't fit how you work:

- It's managed only through the Incus API, with no shell, so there's nothing for Ventoy or Ansible to configure.
- It's built around ZFS, and it refuses loop-backed pools.
- It follows a weekly rolling release.

It's excellent for appliance-style deployments. It's a poor match for your keep-control, Ansible-managed approach.

**One thing to plan for in your firewall work.** Incus manages its own nftables rules for its bridges (7.0 dropped iptables support entirely). Your bootstrap `nftables.conf` drops all forwarded traffic and flushes the whole ruleset, so on an Incus host it will fight Incus. The Foomuuri role for this host needs:

- a zone for `incusbr0`
- DHCP and DNS from instances to the host allowed
- forwarding rules for instance traffic
- no `flush ruleset` anywhere

Want me to add a Debian 13 Incus-host variant to the Ventoy documents? It would cover the interactive partitioning note for a dedicated storage partition, and the firewall adjustments for the bootstrap file.

Sources:

- [Incus 7.0 LTS released (Linuxiac)](https://linuxiac.com/incus-7-0-lts-container-and-virtual-machine-manager-released/)
- [Incus 7.0 LTS has been released (Linux Containers forum)](https://discuss.linuxcontainers.org/t/incus-7-0-lts-has-been-released/26641)
- [Incus (Debian Wiki)](https://wiki.debian.org/Incus)
- [Zabbly Incus packages](https://github.com/zabbly/incus)
- [IncusOS: a new immutable Linux system built for running Incus (Linuxiac)](https://linuxiac.com/incusos-is-a-new-immutable-linux-system-built-for-running-incus/)
- [IncusOS documentation](https://github.com/lxc/incus-os)
- [Incus 7.4 adds near-live container migration (Linuxiac)](https://linuxiac.com/?p=218426)
