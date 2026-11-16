---
title: "Contracts"
version: "0.1.0"
date: 2026-11-15
author: "Christopher Steel"
status: "Draft, verified from the templates and packaged unit files, not yet confirmed on installed nodes"
description: "The contracts uc-ventoy takes part in, which targets implement which contract version, and how each clause is met."
---

# Contracts

uc-ventoy takes part in one contract.

| Contract | Version | Role | Authoritative text |
| --- | --- | --- | --- |
| raw-node | 0.1.0 | Egress (producer) | [uc-contracts](https://github.com/steelcj/uc-contracts), `en/docs/raw-node/uc-raw-node-contract.md` |

The contract is the agreement between raw node producers and the Ansible controller. It says what a raw node provides at handover and what its confirmation reports. uc-ventoy is one producer. It does not change the contract, and where a clause cannot be met the question goes back to uc-contracts rather than being worked around here.

[raw-node-baseline.md](raw-node-baseline.md) describes how uc-ventoy builds a raw node. Its requirements RN-01 to RN-12 are uc-ventoy's own and are wider than the contract. The contract's clauses use longer identifiers such as RN-FW-01, so the two never collide.

## Versions

Two versions are recorded on every node, in `/etc/raw-node/raw-node.conf`, and repeated in the confirmation:

| Value | Set in `bin/build` | On the node | In the confirmation |
| --- | --- | --- | --- |
| Contract version | `CONTRACT_VERSION` | `RAW_NODE_CONTRACT` | `contract` |
| Baseline version | `BASELINE_VERSION` | `RAW_NODE_BASELINE` | `baseline` |

Both are global in `bin/build`, so every target is built to the same contract version. A target must not be built under a contract version it does not implement.

## Targets

| Target | Contract | Baseline | Foomuuri | Status |
| --- | --- | --- | --- | --- |
| `debian/13` | 0.1.0 | 0.3.0 | 0.27 | Template updated, not yet installed with this baseline |
| `ubuntu/24.04` | 0.1.0 | 0.3.0 | 0.22, from `universe` | Template updated, not yet installed |
| `ubuntu/26.04` | 0.1.0 | 0.3.0 | 0.31, from `universe` | Template updated, not yet installed |

Debian 13 passed on metal on 2026-11-15 with baseline 0.2.0, which had the static nftables firewall and the short confirmation. That result does not carry over to contract 0.1.0.

## Shared parameters

The contract's shared parameters are the inputs in `raw-node-input.yml`. The Ansible controller must be given the same values.

| Contract parameter | Input key | Default |
| --- | --- | --- |
| `initial_username` | `initial_username` | None, required |
| `ssh_public_key` | `ssh_public_key` | None, required |
| `ssh_port` | `ssh_port` | 22 |
| `ssh_sources` | `ssh_sources` | Empty, meaning any |

## Clause coverage

Each cell says whether the clause is met and how, or that it needs an install to confirm. Nothing below has been confirmed on a node installed with baseline 0.3.0. "Met" means the templates, the shared files and the offline tests in [the session record](sessions/2026-11-15--20-50-23-session.md) show it.

| Clause | `debian/13` | `ubuntu/24.04` | `ubuntu/26.04` |
| --- | --- | --- | --- |
| RN-NET-01 | Met. `netcfg/hostname` sets `deb13-vt`, and first boot appends the MAC suffix. | Met. `identity.hostname` sets `ub2404-vt`, Subiquity sets `preserve_hostname`, and first boot appends the suffix. | Met. As 24.04, with `ub2604-vt`. |
| RN-ACC-01 | Met. The installer creates the home from `/etc/skel`, and `late_command` adds only `.ssh/authorized_keys`. | Needs an install to confirm. Subiquity creates the home and cloud-init writes `.ssh/authorized_keys` at first boot. Neither is known to add anything else. | Needs an install to confirm. As 24.04. |
| RN-ACC-02 | Met. Key in `authorized_keys`, mode `0600` in a `0700` directory, `PubkeyAuthentication yes`. | Needs an install to confirm. `ssh.authorized-keys`, installed by cloud-init at first boot. | Needs an install to confirm. As 24.04. |
| RN-ACC-03 | Met. `PasswordAuthentication yes` in `10-local-auth.conf`. | Needs an install to confirm. `10-local-auth.conf` sorts before cloud-init's `50-cloud-init.conf`, and `allow-pw: true` makes cloud-init write `yes` as well. | Needs an install to confirm. As 24.04. |
| RN-ACC-04 | Met. `sudo` installed, user added to `sudo`, no `NOPASSWD` rule. | Needs an install to confirm. The default groups include `sudo`, and Subiquity passes no `sudo` rule to cloud-init. | Needs an install to confirm. As 24.04, with sudo-rs as the default sudo. |
| RN-ACC-05 | Met. `passwd/root-password-crypted`, and `PermitRootLogin no`. | Needs an install to confirm. `usermod --password` in `late-commands`, and `PermitRootLogin no`. | Needs an install to confirm. As 24.04. |
| RN-SSH-01 | Met for port 22. Needs an install to confirm for another port. `Port` in `20-raw-node-port.conf`, read by `ssh.service`. | Met in a test of the packaged `sshd-socket-generator`, which gave `ssh.socket` the port from the drop-in. Needs an install to confirm. | Met in the same test with the 26.04 generator. Needs an install to confirm. |
| RN-SSH-02 | Needs an install to confirm. The firewall opens `ssh_port` to `ssh_sources` on `en*`, `eth*` and `wl*`. | Needs an install to confirm. As Debian. | Needs an install to confirm. As Debian. |
| RN-FW-01 | Needs an install to confirm. No other firewall manager is installed, and `nftables.service` is masked. | Needs an install to confirm. `ufw.service` and `nftables.service` are masked. | Needs an install to confirm. As 24.04. |
| RN-FW-02 | Met. Foomuuri 0.27 creates and removes only `table inet foomuuri`, and the only unit that flushes the ruleset is masked. | Met. As Debian, with Foomuuri 0.22. | Met. As Debian, with Foomuuri 0.31. |
| RN-FW-03 | Met on `en*`, `eth*` and `wl*`, from the ruleset Foomuuri 0.27 generates. See the notes below. | Met on the same interfaces, with 0.22. | Met on the same interfaces, with 0.31. |
| RN-FW-04 | Needs an install to confirm. `foomuuri.service` is ordered before `network-pre.target` and ifupdown after it. No ordering cycle in an offline check. | Needs an install to confirm. The packaged unit forms an ordering cycle with cloud-init. The drop-in removes it in an offline check. | Needs an install to confirm. No cycle in an offline check, with or without the drop-in. |
| RN-FWM-01 | Met. The package ships `/etc/foomuuri` empty, and the answer file writes only `00-raw-node.conf`. | Met. As Debian. A systemd drop-in for `foomuuri.service` is also written, see the notes. | Met. As 24.04. |
| RN-REP-01 | Needs an install to confirm. `raw-node-firstboot` writes the file with mode `0644`. The format is checked by `bin/build` on every run. | Needs an install to confirm. As Debian. | Needs an install to confirm. As Debian, with uutils `date` and sudo-rs. |

### Notes on the firewall clauses

**RN-FW-03.** The ruleset was generated by each target's Foomuuri version from the rendered file and read rule by rule:

- Inbound on the public zone: ping, DHCP and DHCPv6 replies, SSH as in RN-SSH-02, replies to connections the node opened, and the ICMP and ICMPv6 types that Foomuuri treats as necessary for networking (errors, neighbour and router discovery, multicast listener messages). Foomuuri 0.27 and 0.31 also accept IGMP queries sent to 224.0.0.1.
- Outbound to the public zone: everything.
- Forwarded: new connections are dropped. Foomuuri's forward chain passes ICMP error messages before that drop, which has no effect because the kernel does not forward on a raw node.
- The public zone is the interfaces named `en*`, `eth*` and `wl*`. An interface with another name, such as `ww*`, `ib*`, a bridge or a bond, is in no zone: nothing is accepted from it, and traffic the node sends through it is rejected. Foomuuri's catch-all `*` would cover every name, but it needs 0.33 or later and fails `nft` on all three packaged versions.
- A reverse path filter, which Foomuuri adds by default, also drops packets that arrive on an interface the node would not use to answer them.

**RN-FW-04.** `foomuuri.service` is ordered before `network-pre.target` in all three versions, and at first boot it is this unit that loads the rules, because `foomuuri-boot.service` only runs once a known good ruleset has been saved.

- `foomuuri-boot.service` exists in 0.27 and 0.31. It is a static unit with no install section, started by `foomuuri.service` itself, so the answer files enable only `foomuuri.service`. Enabling `foomuuri-boot.service` would do nothing on those targets and would fail on 24.04, where the unit does not exist.
- On Ubuntu 24.04 the packaged `foomuuri.service` cannot be used as it is. It keeps systemd's default dependencies, which order it after `sysinit.target`, while cloud-init's network stage is ordered before `sysinit.target` and after `systemd-networkd-wait-online.service`. That is a cycle: Foomuuri, `network-pre.target`, networkd, wait-online, cloud-init, `sysinit.target`, Foomuuri. `systemd-analyze verify` on the packaged unit files reports it and drops `cloud-init.service`, the stage that installs the SSH key. [`shared/systemd/foomuuri-raw-node.conf`](../../shared/systemd/foomuuri-raw-node.conf), written as `/etc/systemd/system/foomuuri.service.d/10-raw-node.conf`, turns the default dependencies off, as `ufw.service` and `nftables.service` do, and the cycle is gone.
- On Ubuntu 26.04 cloud-init no longer orders itself after wait-online, so there is no cycle. It can still wait for the network itself while `sysinit.target` is held back, which would block Foomuuri in the same way, so the same drop-in is written.
- Debian 13 has no cloud-init, the offline check finds no cycle, and the packaged unit is used unchanged.

**RN-FWM-01.** The drop-in on Ubuntu changes when systemd starts Foomuuri, not what Foomuuri does, and `/etc/foomuuri` holds one file on every target. It is how uc-ventoy meets RN-FW-04 on Ubuntu, so it is a matter for this producer and not for the contract, which does not say how a clause is met. It is explained where it happens, in [raw-node-baseline.md](raw-node-baseline.md#the-foomuuri-drop-in-on-ubuntu). Decided on 2026-11-15.

## Observations for uc-contracts

These came up while implementing the confirmation. The report follows the contract's format exactly, so they are open points for the contract, not deviations.

- **Plain MAC addresses can change type.** A MAC whose six parts are all decimal digits below 60, such as the QEMU address `52:54:00:12:34:56`, is read by YAML 1.1 parsers, including PyYAML and therefore Ansible, as a base 60 integer (41135085296) instead of a string. A hexadecimal letter anywhere in the address avoids it. Quoting the value in the confirmation would remove the problem.
- **A `machine_id` of digits only** is read as an integer for the same reason. It is rare, but it loses leading zeros when it happens.
- **Outbound traffic and interface names.** RN-FW-03 says everything outbound is accepted. With the interface patterns that Foomuuri 0.22 to 0.31 need, that holds for `en*`, `eth*` and `wl*` only.

## Checks on an installed node

Run these on one node of each target before treating the target as implementing the contract. Replace `NODE` with an address from the confirmation and `initial` with `initial_username`.

On the node, at the console or over SSH:

```sh
# RN-REP-01, and a first look at everything else
cat /var/lib/raw-node/confirmation.yml
stat -c '%a %U' /var/lib/raw-node/confirmation.yml
systemctl status raw-node-firstboot.service
python3 -c 'import yaml,sys; print(yaml.safe_load(open("/var/lib/raw-node/confirmation.yml")))'

# RN-NET-01
hostnamectl --static

# RN-ACC-01: before the first login if possible, from the console as root
ls -la /home/initial /home/initial/.ssh

# RN-ACC-04, RN-ACC-05
sudo -k; sudo true
sudo passwd -S root
sudo sshd -T | grep -E '^(port|permitrootlogin|passwordauthentication|pubkeyauthentication) '

# RN-SSH-01
sudo ss -ltnp | grep -E 'sshd|systemd'

# RN-FW-01, RN-FW-02, RN-FWM-01
sudo nft list tables
systemctl is-enabled nftables.service foomuuri.service
systemctl is-active foomuuri.service
ls -la /etc/foomuuri
sudo iptables-legacy-save 2>/dev/null | grep -c '^-A'

# RN-FW-03
sudo nft list table inet foomuuri

# RN-FW-04: after a reboot
systemd-analyze critical-chain network-pre.target
journalctl -b | grep -i -E 'ordering cycle|deleted to break'
journalctl -b -u foomuuri.service -u foomuuri-boot.service
```

Expected: `confirmation.yml` has mode `644`, `firewall: foomuuri`, `firewall_active: true`, `sudo: true` and the right `ssh_port`. `passwd -S root` reports `P`. `nft list tables` shows only `table inet foomuuri`. `nftables.service` is `masked`. `/etc/foomuuri` holds only `00-raw-node.conf`. The `iptables-legacy-save` count is 0 or the command does not exist. The journal shows no ordering cycle, and `foomuuri.service` started before the network units.

From another machine inside `ssh_sources`:

```sh
# RN-ACC-02
ssh -p PORT -o PreferredAuthentications=publickey initial@NODE true
# RN-ACC-03
ssh -p PORT -o PreferredAuthentications=password -o PubkeyAuthentication=no initial@NODE true
# RN-ACC-05: must be refused
ssh -p PORT -o PreferredAuthentications=password -o PubkeyAuthentication=no root@NODE true
# RN-FW-03
ping -c 2 NODE
nmap -Pn -p 1-1024 NODE
```

If `ssh_sources` is set, repeat the first command from a machine outside the list. It must time out.

Per target:

| Target | Also check |
| --- | --- |
| `debian/13` | The installer accepts the answer file: its `late_command` is about twice as long as the one that passed on metal. `journalctl -b -u networking.service` starts after `foomuuri.service`. |
| `ubuntu/24.04` | The installer could reach the archive for `foomuuri`, which is in `universe`. `systemctl cat foomuuri.service` shows the `10-raw-node.conf` drop-in. `cloud-init status --long` reports `done` with no errors, and `/home/initial/.ssh/authorized_keys` exists. `systemctl is-enabled ufw.service` reports `masked`. `systemctl cat ssh.socket` shows the port when it is not 22. The hostname suffix survives a reboot. |
| `ubuntu/26.04` | As 24.04. In addition `sudo: true` in the confirmation, which is measured through sudo-rs, and a `timestamp` with an offset, which comes from uutils `date`. |
