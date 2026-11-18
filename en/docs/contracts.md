---
title: "Contracts"
version: "0.2.0"
date: 2026-11-17
author: "Christopher Steel"
status: "Draft, verified from the templates, the packaged files and the package sources of each target, not yet confirmed on installed nodes"
description: "The contracts uc-ventoy takes part in, which targets implement which contract version, and how each clause is met."
---

# Contracts

uc-ventoy takes part in one contract.

| Contract | Version | Role | Authoritative text |
| --- | --- | --- | --- |
| raw-node | 0.3.0 | Egress (producer) | [uc-contracts](https://github.com/steelcj/uc-contracts), `en/docs/node-onboarding-pipeline/1-raw-node-egress/uc-raw-node-contract.md`, as of commit `0a0fe9d` |

The contract is the agreement between raw node producers and the Ansible controller. It says what a raw node provides at handover and what its confirmation reports. uc-ventoy is one producer. It does not change the contract, and where a clause cannot be met the question goes back to uc-contracts rather than being worked around here.

The reasons behind version 0.3.0 are in uc-contracts as well: `en/docs/decisions/uc-privileged-access-decision-v0.3.0.md`, and the privileged access policy it points to. The policy governs normalized nodes, not raw nodes. The contract applies the parts of it that a raw node can already meet.

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
| `debian/13` | 0.3.0 | 0.4.0 | 0.27 | Template updated, not yet installed with this baseline |
| `ubuntu/24.04` | 0.3.0 | 0.4.0 | 0.22, from `universe` | Template updated, not yet installed |
| `ubuntu/26.04` | 0.3.0 | 0.4.0 | 0.31, from `universe` | Template updated, not yet installed |

Debian 13 passed on metal on 2026-11-15 with baseline 0.2.0, which had the static nftables firewall and the short confirmation. On 2026-11-17 the operator reported a successful Debian 13 install from the stick as written on 2026-11-16, which held baseline 0.3.0 for contract 0.1.0. Neither result carries over to contract 0.3.0.

### Version history

| Contract | Baseline | What changed for uc-ventoy |
| --- | --- | --- |
| 0.1.0 | 0.3.0 | First contract: Foomuuri firewall, the confirmation with interfaces and identity |
| 0.2.0 | Not built | RN-FWM-01 retired: the contract no longer names a firewall manager. Foomuuri stays, as this producer's choice, and the firewall clauses RN-FW-01 to RN-FW-04 are unchanged. No baseline was built for 0.2.0 alone. |
| 0.3.0 | 0.4.0 | Shared parameter `initial_password`. RN-ACC-03 (password login over SSH) retired. RN-ACC-06, RN-ACC-07, RN-ACC-08 and RN-SSH-03 added. Confirmation fields `sudo_nopasswd`, `ssh_password_auth` and `ssh_host_keys` added. |

## Shared parameters

The contract's shared parameters are the inputs in `raw-node-input.yml`, except `initial_password`, which uc-ventoy never sees. The Ansible controller must be given the same values.

| Contract parameter | Input key | Default |
| --- | --- | --- |
| `initial_username` | `initial_username` | None, required |
| `initial_password` | None. Its hash is `initial_password_hash`. | None, required. Held by the controller, see below. |
| `ssh_public_key` | `ssh_public_key` | None, required |
| `ssh_port` | `ssh_port` | 22 |
| `ssh_sources` | `ssh_sources` | Empty, meaning any |

### The initial password

The contract makes `initial_password` the controller's secret: ingress holds it in its secret store, and the producer receives only its hash. Nodes are built before their hostnames exist, so one password serves every node of a build batch. Onboarding later replaces it with a password unique to each node.

The controller's side is `scripts/new_raw_batch.py` in uc-controller. **That script is not built yet.** As the kickoff for contract 0.3.0 describes it, it:

1. creates a batch password,
2. keeps it in the controller's secret store,
3. prints the batch ID and a SHA-512 crypt hash of the password.

The operator copies the hash into `raw-node-input.yml` as `initial_password_hash`, and may copy the batch ID as `password_batch`. uc-ventoy has no way of entering the plain password, and none is to be added. `root_password_hash` is unaffected: the `root` password belongs to this producer, is made with `mkpasswd --method=sha-512`, and the controller never needs it.

Until the script exists, a batch can be made by hand on the workstation:

```sh
# A 24 character password; keep it in the password manager under the batch ID
python3 -c "import secrets,string; print(''.join(secrets.choice(string.ascii_letters+string.digits) for _ in range(24)))"
# Its hash for raw-node-input.yml; mkpasswd asks for the password
mkpasswd --method=sha-512
```

```yaml
initial_password_hash: "$6$..."
password_batch: "raw-2026-11-17-manual"
```

`password_batch` is not a contract field and is not in the confirmation. `bin/build` writes it to `/etc/raw-node/raw-node.conf` as `RAW_NODE_PASSWORD_BATCH`, so that an operator at the console can tell which batch password a node was built with. It accepts letters, digits, dots, hyphens and underscores, at most 64, because the node's scripts read that file with the shell.

**RN-ACC-08** says `initial_password` is at least 15 characters. `bin/build` sees only the hash and cannot check the length, so uc-ventoy meets this clause by procedure: the password comes from the controller's generator, or from the 24-character command above. A hash made from any other password would break the clause without `bin/build` noticing.

## Clause coverage

Each cell says whether the clause is met and how, or that it needs an install to confirm. Nothing below has been confirmed on a node installed with baseline 0.4.0. "Met" means the templates, the shared files and the offline checks in [the session record](sessions/2026-11-17--21-40-33-session.md) show it. The offline checks read the packages of each target at these versions:

| Package | `debian/13` | `ubuntu/24.04` | `ubuntu/26.04` |
| --- | --- | --- | --- |
| OpenSSH server | 1:10.0p1-7+deb13u4 | 1:9.6p1-3ubuntu13.20 | 1:10.2p1-2ubuntu3.8 |
| sudo | sudo 1.9.16p2-3+deb13u2 | sudo 1.9.15p5-3ubuntu5.24.04.4 | sudo-rs 0.2.13-0ubuntu1.2, with `/etc/sudoers` from sudo-common 1.2ubuntu |
| Installer | Debian Installer | Subiquity 24.04.5 | Subiquity 26.04.1 |
| cloud-init | None | 25.1.4 (security) and 26.2 (updates) | 26.2 |

| Clause | `debian/13` | `ubuntu/24.04` | `ubuntu/26.04` |
| --- | --- | --- | --- |
| RN-NET-01 | Met. `netcfg/hostname` sets `deb13-vt`, and first boot appends the MAC suffix. | Met. `identity.hostname` sets `ub2404-vt`, Subiquity sets `preserve_hostname`, and first boot appends the suffix. | Met. As 24.04, with `ub2604-vt`. |
| RN-ACC-01 | Met. The installer creates the home from `/etc/skel`, and `late_command` adds only `.ssh/authorized_keys`. | Needs an install to confirm. Subiquity creates the home and cloud-init writes `.ssh/authorized_keys` at first boot. Neither is known to add anything else. | Needs an install to confirm. As 24.04. |
| RN-ACC-02 | Met. Key in `authorized_keys`, mode `0600` in a `0700` directory, `PubkeyAuthentication yes`. | Needs an install to confirm. `ssh.authorized-keys`, installed by cloud-init at first boot. | Needs an install to confirm. As 24.04. |
| RN-ACC-03 | Retired in contract 0.3.0. Password login over SSH is now refused, see RN-SSH-03. | Retired. | Retired. |
| RN-ACC-04 | Met. `sudo` installed, the user added to the `sudo` group by `late_command`, and no rule without a password (RN-ACC-07), so sudo asks for the password. | Needs an install to confirm. Subiquity adds the user to `sudo`, and no rule without a password. | Needs an install to confirm. As 24.04, through sudo-rs. |
| RN-ACC-05 | Met. `passwd/root-password-crypted`, and `PermitRootLogin no`. | Needs an install to confirm. `usermod --password` in `late-commands`, and `PermitRootLogin no`. | Needs an install to confirm. As 24.04. |
| RN-ACC-06 | Met, needs an install to confirm. `passwd/user-password-crypted` sets the password, and the console login goes through getty and PAM, which the answer file does not touch. No serial console is configured. | Needs an install to confirm. `identity.password` sets the password, and Subiquity passes `lock_passwd: false` to cloud-init, so it is not locked. getty and PAM untouched, no serial console. | Needs an install to confirm. As 24.04. |
| RN-ACC-07 | Met. The stock `/etc/sudoers` and `/etc/sudoers.d/README` have no `NOPASSWD` or `!authenticate`, and the answer file writes no sudoers file. | Met in the package sources, needs an install to confirm. Stock files as Debian. Subiquity passes cloud-init the user without a `sudo` key and without the `default` user, so cloud-init writes no `/etc/sudoers.d/90-cloud-init-users`. | Met in the package sources, needs an install to confirm. As 24.04. sudo-rs reads `/etc/sudoers-rs` when it exists and `/etc/sudoers` otherwise. No package ships `/etc/sudoers-rs`, and sudo-common's `/etc/sudoers` has no `NOPASSWD`. |
| RN-ACC-08 | Met by procedure. See [The initial password](#the-initial-password). | Met by procedure. | Met by procedure. |
| RN-SSH-01 | Met for port 22. Needs an install to confirm for another port. `Port` in `20-raw-node-port.conf`, read by `ssh.service`. | Met in a test of the packaged `sshd-socket-generator`, which gave `ssh.socket` the port from the drop-in. Needs an install to confirm. | Met in the same test with the 26.04 generator. Needs an install to confirm. |
| RN-SSH-02 | Needs an install to confirm. The firewall opens `ssh_port` to `ssh_sources` on `en*`, `eth*` and `wl*`. | Needs an install to confirm. As Debian. | Needs an install to confirm. As Debian. |
| RN-SSH-03 | Met. `10-local-auth.conf` sets `PasswordAuthentication no`, `KbdInteractiveAuthentication no` and `PermitEmptyPasswords no`. The stock `sshd_config` includes the drop-ins before any setting, and this target's `sshd -G` gives `no` for all three. | Met in an offline check, needs an install to confirm. As Debian, and `allow-pw: false` makes cloud-init write `PasswordAuthentication no` to `50-cloud-init.conf`. This target's `sshd -G` gives `no` even when that file says `yes`. cloud-init runs only at the first boot, see the notes. | Met in an offline check, needs an install to confirm. As 24.04. |
| RN-FW-01 | Needs an install to confirm. No other firewall manager is installed, and `nftables.service` is masked. | Needs an install to confirm. `ufw.service` and `nftables.service` are masked. | Needs an install to confirm. As 24.04. |
| RN-FW-02 | Met. Foomuuri 0.27 creates and removes only `table inet foomuuri`, and the only unit that flushes the ruleset is masked. | Met. As Debian, with Foomuuri 0.22. | Met. As Debian, with Foomuuri 0.31. |
| RN-FW-03 | Met on `en*`, `eth*` and `wl*`, from the ruleset Foomuuri 0.27 generates. See the notes below. | Met on the same interfaces, with 0.22. | Met on the same interfaces, with 0.31. |
| RN-FW-04 | Needs an install to confirm. `foomuuri.service` is ordered before `network-pre.target` and ifupdown after it. No ordering cycle in an offline check. | Needs an install to confirm. The packaged unit forms an ordering cycle with cloud-init. The drop-in removes it in an offline check. | Needs an install to confirm. No cycle in an offline check, with or without the drop-in. |
| RN-FWM-01 | Retired in contract 0.2.0. Foomuuri stays, as this producer's choice, see the notes. | Retired. | Retired. |
| RN-REP-01 | Needs an install to confirm. `raw-node-firstboot` writes the file with mode `0644`. The format is checked by `bin/build` on every run. | Needs an install to confirm. As Debian. | Needs an install to confirm. As Debian, with uutils `date` and sudo-rs. |

### Confirmation fields

`bin/build` checks every field's presence, order and type on every run, by running the report on the workstation with a test folder in place of `/etc`. How each value is measured on a node is in [raw-node-baseline.md](raw-node-baseline.md#first-boot-report). The values on a real node need an install to confirm on every target.

| Field | How | Checked offline |
| --- | --- | --- |
| `contract`, `baseline` | From `raw-node.conf` | Read back from a test `raw-node.conf` |
| `timestamp` | `date --iso-8601=seconds` | Present |
| `machine_id` | `/etc/machine-id`, quoted | An all-digit ID stays a string |
| `user`, `sudo` | `id`, and `sudo -n -l -U` as root | Types |
| `sudo_nopasswd` | Non-comment lines of `/etc/sudoers`, `/etc/sudoers-rs` and every file in `/etc/sudoers.d` | `false` with a commented `NOPASSWD`, `true` with a `NOPASSWD` rule and with `Defaults:initial !authenticate` |
| `ssh_port`, `ssh_password_auth` | `sshd -G`, falling back to `sshd -T` | Types, from the workstation's sshd. With each target's sshd binary: `no` for both settings on the raw node configuration. |
| `ssh_host_keys` | Each `/etc/ssh/ssh_host_*_key.pub`, with `ssh-keygen -l` | The fingerprint of a test key matches `ssh-keygen -l`. Without a key, nothing is printed and the exit status is 2. |
| `firewall`, `firewall_active`, `python` | Unchanged since contract 0.1.0 | Types |
| `interfaces` | Unchanged, except `mac` is quoted | `52:54:00:12:34:56` stays a string |

### Notes on the SSH and sudo clauses

**cloud-init after the first boot.** On both Ubuntu targets Subiquity has cloud-init write `/etc/cloud/cloud-init.disabled` at the end of the first boot, so no cloud-init module runs at a later boot and none can turn password authentication back on. At the first boot, `set_passwords` runs in cloud-init's network stage and writes `50-cloud-init.conf` from `ssh_pwauth`, which Subiquity sets from `allow-pw`. Even if that file said `yes`, `10-local-auth.conf` is read first and sshd keeps the first value it reads.

**Host keys and the report.** The report fails rather than print an empty `ssh_host_keys`. On Ubuntu, cloud-init deletes the host keys made at install time and makes new ones in its network stage, `cloud-init.service` on 24.04 and `cloud-init-network.service` on 26.04. `raw-node-firstboot.service` is ordered after both names. On Debian the keys are made when `openssh-server` is installed, and `sshd-keygen.service` only fills in missing ones at a first boot. The unit is ordered after it too. Should the report still fail, `raw-node-firstboot` writes no confirmation, shows the reason at the console, and runs again at the next boot.

**`ssh_password_auth` and `sshd -T`.** The contract defines the field "as reported by `sshd -T`". `sshd -T` fails on Ubuntu until the first connection has created `/run/sshd`, so the report uses `sshd -G` first, which prints the same settings without that check, and `sshd -T` as the fallback.

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

**Foomuuri as this producer's choice.** Contract 0.2.0 retired RN-FWM-01, so the contract no longer says which firewall manager a raw node runs, and the firewall clauses are all it promises. uc-ventoy keeps Foomuuri with one file, `/etc/foomuuri/00-raw-node.conf`, on every target, and `firewall: foomuuri` in the confirmation says so. The drop-in on Ubuntu changes when systemd starts Foomuuri, not what Foomuuri does. It is how uc-ventoy meets RN-FW-04 on Ubuntu, explained in [raw-node-baseline.md](raw-node-baseline.md#the-foomuuri-drop-in-on-ubuntu).

## Observations for uc-contracts

These came up while implementing the contract. uc-ventoy follows the numbered clauses and the field table, so they are open points for the contract, not deviations.

- **Unfinished text in the contract.** Three passages of contract 0.3.0 at commit `0a0fe9d` read as notes rather than contract text:
  - Under the introduction, a bullet: "adding a line to the raw node contract that `mac` and `machine_id` are quoted strings, so every future producer does the same." The format example still shows both unquoted. uc-ventoy quotes them, which any YAML parser reads as the same string.
  - Under Firewall, before the clause table: "The raw node has a firewall that allows incoming communications to take place over port 22" and "Below needs to be rewritten to reflect that". Port 22 is the default of the shared parameter `ssh_port`, not a fixed value, and the clauses below it are unchanged. uc-ventoy follows RN-SSH-01, RN-SSH-02 and RN-FW-01 to RN-FW-04 as numbered.
  - Under Firewall manager: "Foomuuri is installed during onboarding, which comes after this contract." uc-ventoy already installs Foomuuri on the raw node, so onboarding will find it there.
- **Other values that change type.** The quoting of `mac` and `machine_id` closes the two cases found before. Two others remain, in the contract's own format: `timestamp` is read by YAML 1.1 parsers, including PyYAML and therefore Ansible, as a datetime, not a string; and an IPv6 address whose groups are all decimal and below 60, with no `::`, such as `2001:12:34:56:12:34:56:12`, is read as a base 60 integer. Quoting them would remove both.
- **`sudo_nopasswd` and comments.** The kickoff for 0.3.0 asked for `true` when any sudoers file "contains" `NOPASSWD` or `!authenticate`. The contract says "rule". uc-ventoy follows the contract and skips comment lines, because the stock `sudoers.dist` examples and many hand-edited files carry commented `NOPASSWD` lines. It reads every file in `/etc/sudoers.d`, including those sudo itself skips, so a backup such as `90-foo~` still counts.
- **Outbound traffic and interface names.** RN-FW-03 says everything outbound is accepted. With the interface patterns that Foomuuri 0.22 to 0.31 need, that holds for `en*`, `eth*` and `wl*` only.

Removed on 2026-11-17, because the report now quotes the values: the observations that a MAC address of decimal digits and a `machine_id` of digits only are read as integers.

## Checks on an installed node

Run these on one node of each target before treating the target as implementing the contract. Replace `NODE` with an address from the confirmation, `PORT` with `ssh_port`, and `initial` with `initial_username`.

At the console, before anything else:

```sh
# RN-ACC-06: log in as initial with the batch password at the tty1 login prompt.
# The prompt also shows the confirmation (RN-REP-01).
```

On the node, at the console or over SSH with the key:

```sh
# RN-REP-01, and a first look at everything else
cat /var/lib/raw-node/confirmation.yml
stat -c '%a %U' /var/lib/raw-node/confirmation.yml
systemctl status raw-node-firstboot.service
python3 -c 'import yaml,sys; print(yaml.safe_load(open("/var/lib/raw-node/confirmation.yml")))'
grep RAW_NODE /etc/raw-node/raw-node.conf

# RN-NET-01
hostnamectl --static

# RN-ACC-01: before the first login if possible, from the console as root
ls -la /home/initial /home/initial/.ssh

# RN-ACC-04: must ask for the password, and accept the batch password
sudo -k; sudo true
# RN-ACC-05
sudo passwd -S root

# RN-ACC-07: must print nothing, or comment lines only
sudo grep -r -n -E 'NOPASSWD|!authenticate' /etc/sudoers /etc/sudoers.d
ls -la /etc/sudoers-rs 2>/dev/null

# RN-SSH-03, RN-ACC-05
sudo sshd -T | grep -E '^(port|permitrootlogin|pubkeyauthentication|passwordauthentication|kbdinteractiveauthentication|permitemptypasswords) '
ls /etc/ssh/sshd_config.d/

# ssh_host_keys: must match the confirmation
for f in /etc/ssh/ssh_host_*_key.pub; do ssh-keygen -l -f "$f"; done

# RN-SSH-01
sudo ss -ltnp | grep -E 'sshd|systemd'

# RN-FW-01, RN-FW-02
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

Expected: `confirmation.yml` has mode `644`, `contract: 0.3.0`, `baseline: 0.4.0`, `sudo: true`, `sudo_nopasswd: false`, `ssh_password_auth: false`, the right `ssh_port`, `firewall: foomuuri` and `firewall_active: true`, and the Python line prints `machine_id` and every `mac` as strings. `raw-node.conf` shows the batch ID when `password_batch` was set. `sudo true` asks for the password. `passwd -S root` reports `P`. The `grep` finds no rule, and `/etc/sudoers-rs` does not exist. `sshd -T` reports `permitrootlogin no`, `pubkeyauthentication yes`, and `no` for the other three. The fingerprints are those in `ssh_host_keys`, one entry per key. `nft list tables` shows only `table inet foomuuri`. `nftables.service` is `masked`. `/etc/foomuuri` holds only `00-raw-node.conf`. The `iptables-legacy-save` count is 0 or the command does not exist. The journal shows no ordering cycle, and `foomuuri.service` started before the network units.

From another machine inside `ssh_sources`:

```sh
# Host keys: the fingerprints must be those in ssh_host_keys
ssh-keyscan -p PORT NODE 2>/dev/null | ssh-keygen -l -f -
# RN-ACC-02: must log in
ssh -p PORT -o PreferredAuthentications=publickey initial@NODE true
# RN-SSH-03: both must be refused with "Permission denied (publickey)",
# without a password prompt
ssh -p PORT -o PreferredAuthentications=password,keyboard-interactive -o PubkeyAuthentication=no initial@NODE true
ssh -p PORT -o PreferredAuthentications=password,keyboard-interactive -o PubkeyAuthentication=no root@NODE true
# RN-ACC-04 over SSH: must ask for the password
ssh -t -p PORT initial@NODE 'sudo -k; sudo true'
# RN-FW-03
ping -c 2 NODE
nmap -Pn -p 1-1024 NODE
```

If `ssh_sources` is set, repeat the key login from a machine outside the list. It must time out.

Per target:

| Target | Also check |
| --- | --- |
| `debian/13` | The installer accepts the answer file: its `late_command` has grown again with the longer report. `journalctl -b -u networking.service` starts after `foomuuri.service`. |
| `ubuntu/24.04` | The installer could reach the archive for `foomuuri`, which is in `universe`. `/etc/ssh/sshd_config.d/50-cloud-init.conf` says `PasswordAuthentication no`. `/etc/sudoers.d/90-cloud-init-users` does not exist, and no `ubuntu` user exists. `/etc/cloud/cloud-init.disabled` exists after the first boot. `systemctl cat foomuuri.service` shows the `10-raw-node.conf` drop-in. `cloud-init status --long` reports `done` with no errors, and `/home/initial/.ssh/authorized_keys` exists. `systemctl is-enabled ufw.service` reports `masked`. `systemctl cat ssh.socket` shows the port when it is not 22. The hostname suffix survives a reboot. |
| `ubuntu/26.04` | As 24.04. In addition `sudo: true` in the confirmation and the password prompt of `sudo true`, both through sudo-rs, and a `timestamp` with an offset, which comes from uutils `date`. |
