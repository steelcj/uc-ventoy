# Foomuuri -- Handoff etails Regarding Ventoy Ansible Handoff

> Superseded on 2026-11-15. These notes describe the earlier design, in which the answer files shipped a static `nftables.conf` and Ansible installed Foomuuri at handover. Since baseline 0.3.0 every raw node, Ubuntu included, ships Foomuuri from the start, as the raw node contract requires. See [contracts.md](../contracts.md) and [raw-node-baseline.md](../raw-node-baseline.md#firewall). The notes are kept as a record of the reasoning.

## Do not use Foomuuri on Ubuntu Server

**Ventoy and the answer files: bootstrap only.** Install the OS and create the `initial` account with your SSH key, plus just enough protection to make the node reachable and safe until Ansible first runs. That means keeping the minimal bootstrap firewall (SSH in, everything else dropped). Without it, a freshly installed node with SSH password login enabled sits unprotected until someone runs the playbook. These files stay deliberately small, and they’ll always differ a little between Debian and Ubuntu, because the installers differ.

**Ansible: the definitive state.** Foomuuri, the final SSH policy, packages, users and everything else that makes nodes identical. Ansible is where homogeneity comes from, because the same roles run against every node whatever its origin. The installer can’t give you that: Debian preseed and Ubuntu autoinstall are different mechanisms, Foomuuri versions differ by release, and Ansible keeps enforcing the state after day one, which an installer never does.

**What this changes in the documents:**

- The SSH drop-in and `nftables.conf` are labelled as bootstrap, to be taken over by Ansible.
- Anything not needed to reach the node and run Ansible is a candidate to move out of the answer files.
- Optionally, you can tighten the bootstrap further: start with key-only SSH, and let Ansible turn on password authentication once Foomuuri is in place. That makes the window between first boot and the first Ansible run smaller.

Rule of thumb: if Ansible would have to manage a setting anyway, it doesn’t belong in the answer file.

Want me to issue v0.1.2 of both documents framed that way, with the 24.04.5 ISO filename update included?



Ventoy won't fight you. It doesn't create a firewall itself, it only passes the answer file to the installer. The `nftables` rules come from our preseed and autoinstall files, and Ansible can replace them with Foomuuri. Two things need care: the handover itself, and package versions.

**Keep the installer firewall as a bootstrap.** I'd keep the plain `nftables.conf` in the answer files even though Ansible will replace it. The nodes come up with SSH password authentication enabled, so something should filter traffic between first boot and the first Ansible run. Treat it as the bootstrap firewall that Ansible takes over from, not as the long-term one.

**The handover.** The main trap is `nftables.service`. Our `/etc/nftables.conf` starts with `flush ruleset`, which will clear the entire nftables configuration, including all tables, chains and rules, sets or maps. That clears Foomuuri's rules too. If the service stays enabled, it can wipe Foomuuri's rules at boot or on a restart. The same problem has been reported elsewhere: nftables.service flushes entire rule set when reloaded or stopped. A safe order for the play:

1. Install `foomuuri`, deploy the config templates, and run `foomuuri check` before applying anything.
2. Enable and start Foomuuri, then confirm SSH still connects.
3. Remove the bootstrap table with `nft delete table inet filter`, and run `systemctl disable nftables`. Optionally replace `/etc/nftables.conf` with a stub that has no `flush ruleset`, so a stray restart can't do damage.
4. Reboot once and check that `nft list tables` shows only Foomuuri's table.

The order in steps 2 and 3 matters. While both tables exist, a packet has to be accepted by both, so the overlap is stricter, not looser. You only lose access if Foomuuri's own config drops SSH. As a safety net for the first rollout, schedule a timed rollback before the cutover, for example `systemd-run --on-active=5m` re-enabling the bootstrap rules, and cancel it once Ansible reconnects. Or do the first run on an Incus VM where you have console access.

**Versions across your targets.** Ubuntu 24.04 ships foomuuri (0.22-1) [universe], while Debian has 0.31-1~bpo13+1 (main) [trixie-backports] and 0.27-2 (main) [trixie]. That's a big spread for a "same baseline" goal. One Ansible template may not mean the same thing on 0.22 and 0.31. Either pin one version everywhere (backports on Debian, an upstream or self-built package on Ubuntu), or write the config against the oldest version you deploy and test it on each. Being in Ubuntu's universe component also means the 24.04 package gets community rather than Canonical security maintenance. Check that the USN your notes cite actually covers the noble version before relying on it.

**Other notes:**

- **Servers:** skip `foomuuri-firewalld` and the D-Bus service. Your notes point out that's where the 2025 CVEs were, and a server doesn't need NetworkManager switching zones.
- **The Mint desktop:** there, `foomuuri-firewalld` is the useful part, letting NetworkManager move between home and public zones. Remove Mint's `ufw` so only one tool manages the firewall.
- **Docker or Incus:** as I understand it, Foomuuri manages its own table rather than flushing everything, so it should coexist with Docker or Incus better than our bootstrap file does. I couldn't open the Foomuuri docs to confirm, so check `nft list tables` after it starts.
- **The name** is Foomuuri, with an "i" at the end ("muuri" is Finnish for wall). Worth getting right in role names and package lists.

So the Ventoy documents don't need to change. The only addition would be a sentence saying the shipped `nftables.conf` is a bootstrap firewall that the configuration management takes over. Want me to add that to both documents, and sketch the Ansible role layout for the Foomuuri handover?

Sources:

- [Ubuntu security documentation: nftables](https://documentation.ubuntu.com/security/security-features/network/firewall/nftables/)
- [Arch Linux bug 65220: nftables.service flushes entire rule set](https://bugs.archlinux.org/task/65220)
- [Ubuntu packages: foomuuri in noble](https://packages.ubuntu.com/km/noble/foomuuri)
- [Debian sources: foomuuri](https://sources.debian.org/patches/foomuuri)
