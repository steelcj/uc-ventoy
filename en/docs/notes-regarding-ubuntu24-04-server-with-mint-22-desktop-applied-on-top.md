# Notes regarding Ubuntu24.04 server with Mint 22 desktop applied on top

Yes. 24.04.5 is a fine base for a Mint 22 desktop, and it's the best 24.04 ISO to use now.

**Why it works:** 24.04.5 was released on September 10 and is the fifth (and final) point release of Ubuntu 24.04 LTS. Point releases only refresh the installation media. Once the system runs `apt update`, it uses the same `noble` archive that Mint 22 is built on. Linux Mint 22.3 "Zena" (January 2026) runs on Ubuntu 24.04.3 LTS, so .3 versus .5 makes no practical difference. The support window is also the same: standard security maintenance for the Ubuntu 24.04 LTS series continues until May 31, 2029.

**Kernel:** 24.04.5 employs an HWE kernel based on Linux Kernel 7.0, with the kernel branch drawn from Ubuntu 26.04 LTS, and the GA kernel (v6.8 for 24.04) stays available to install. Either works under Mint. For a long-lived machine, the GA kernel (6.8) changes less over time, and the HWE kernel (7.0) supports newer hardware better.

**What to watch when adding Mint on top of the server install:**

- **Networking:** the server install uses netplan with systemd-networkd, while the Mint desktop expects NetworkManager. Unless you set `renderer: NetworkManager` in netplan, the network applet will show the connection as unmanaged.
- **Firewall:** Mint ships `ufw`. Keep it inactive, or remove it, so `nftables` stays the only firewall manager and the baseline doesn't change.
- **Snaps:** Mint blocks `snapd` through an apt preference, and Ubuntu Server has it installed. Expect apt to propose removing `snapd` when the Mint packages go in. Check the list before confirming.
- **cloud-init:** it stays installed from the server install. It's harmless, or you can turn it off with `touch /etc/cloud/cloud-init.disabled` once the system is built.
- **Upgrades:** the result is an unsupported hybrid. Mint's upgrade tool assumes a real Mint install, so moving to Mint 23 (based on 26.04) later will most likely mean rebuilding rather than upgrading.

The node baseline itself (accounts, SSH drop-in, `nftables.conf`) needs no changes. The only edit to the 24.04 document is the ISO filename in the directory tree and `ventoy.json`.

Want me to issue v0.1.2 with the 24.04.5 filename? I can also add an optional "Mint 22 desktop overlay" section covering the repository setup, the netplan switch, and the `ufw` and `snapd` handling.

Sources:

- [Ubuntu 24.04.5 LTS released (The Register)](https://www.theregister.com/a/5296292)
- [Ubuntu 24.04.5 LTS released with Linux kernel 7.0 (OMG! Ubuntu)](https://www.omgubuntu.co.uk/2026/09/ubuntu-24-04-5-lts-released-with-linux-kernel-7-0/amp)
- [Ubuntu 24.04.5 LTS (SecurityOnline)](https://securityonline.info/ubuntu-24-04-5-lts/)
- [Linux Mint vs Ubuntu for Developers (Kunal Ganglani)](https://www.kunalganglani.com/blog/linux-mint-vs-ubuntu-developers.md)
