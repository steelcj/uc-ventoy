---
title: "Ventoy Automated Configuration for Ubuntu 24.04 Server"
version: "0.4.0"
date: 2026-11-17
author: "Christopher Steel"
status: "Draft, reviewed against current documentation, not yet tested in virtual machines"
description: "Ubuntu Server 24.04 LTS autoinstall answer file that Ventoy delivers to Subiquity to produce the raw node defined in raw-node-baseline.md, also usable as a Linux Mint 22 base."
baseline: "raw-node-baseline.md"
---

# Ventoy Automated Configuration for Ubuntu 24.04 Server

```mermaid
flowchart TD
    scope["Scope"] --> stick["Stick layout"]
    stick --> answer["Answer file"]
    answer --> notes["Notes on the autoinstall file"]
    notes --> cover["Requirement coverage"]
    cover --> checks["Ubuntu-specific checks"]
    checks --> mint["Use as a Linux Mint 22 base"]
    mint --> review["Review history"]
    review --> log["Change log"]

    click scope href "#scope" "Scope" _self
    click stick href "#stick-layout" "Stick layout" _self
    click answer href "#answer-file" "Answer file" _self
    click notes href "#notes-on-the-autoinstall-file" "Notes on the autoinstall file" _self
    click cover href "#requirement-coverage" "Requirement coverage" _self
    click checks href "#ubuntu-specific-checks" "Ubuntu-specific checks" _self
    click mint href "#use-as-a-linux-mint-22-base" "Use as a Linux Mint 22 base" _self
    click review href "#review-history" "Review history" _self
    click log href "#change-log" "Change log" _self
```

Some renderers, including GitHub, disable clicks inside Mermaid charts. The same sections are linked here: [Scope](#scope), [Stick layout](#stick-layout), [Answer file](#answer-file), [Notes on the autoinstall file](#notes-on-the-autoinstall-file), [Requirement coverage](#requirement-coverage), [Ubuntu-specific checks](#ubuntu-specific-checks), [Use as a Linux Mint 22 base](#use-as-a-linux-mint-22-base), [Review history](#review-history), [Change log](#change-log), [References](#references).

## Scope

- Ubuntu Server 24.04 LTS "Noble Numbat", installed with Subiquity and cloud-init autoinstall from the 24.04.5 installation media, released on September 10, 2026 <a id="cite-gupta-2026"></a>([Gupta, 2026](#ref-gupta-2026)). Ubuntu Server 24.04 receives maintenance updates for five years from its April 2024 release <a id="cite-jacquet-2026"></a>([Jacquet, 2026](#ref-jacquet-2026)).
- The result is the raw node defined in [raw-node-baseline.md](raw-node-baseline.md), which holds the requirements (RN-01 to RN-12), the inputs, the canonical files, the general acceptance checks and the handover to Ansible. This document holds only what is specific to Ubuntu 24.04.
- The installer hostname is `ub2404-vt`, which becomes `ub2404-vt-xxxx` at first boot.
- A node built from this document can also serve as the base for a Linux Mint 22 desktop. See [Use as a Linux Mint 22 base](#use-as-a-linux-mint-22-base).

## Stick layout

```text
/iso/ubuntu/24.04/ubuntu-24.04.5-live-server-amd64.iso
/answer-files/ubuntu/24.04/user-data
```

`bin/build --stick` writes the answer file and the `ventoy.json` entry, which matches the folder `/iso/ubuntu/24.04` rather than the ISO filename. See [ISO downloads](iso-downloads.md) for fetching and verifying the ISO.

The 24.04.5 server ISO offers both the general availability kernel (6.8) and the hardware enablement (HWE) kernel, which in 24.04.5 is based on Linux 7.0 <a id="cite-larabel-2026"></a>([Larabel, 2026](#ref-larabel-2026)). Either serves the raw node. The general availability kernel changes less over the life of the release, and the HWE kernel supports newer hardware. A host that will run Incus 7.0 needs kernel 6.12 or later <a id="cite-linuxiac-incus-7"></a>([Linuxiac, 2026](#ref-linuxiac-incus-7)), so choose the HWE kernel for that role.

## Answer file

Template: [`ubuntu/24.04/user-data.j2`](../../ubuntu/24.04/user-data.j2). `bin/build` renders it into `build/answer-files/ubuntu/24.04/user-data`, filling in the [inputs](raw-node-baseline.md#inputs) and embedding the [canonical files](raw-node-baseline.md#canonical-files).

The `autoinstall` section sets locale, keyboard and time zone, keeps storage interactive, creates the initial user through `identity`, installs the OpenSSH server with the key, and password authentication off, through `ssh`, and adds `nftables`, `python3` and `foomuuri`. Its `late-commands` write each canonical file with a quoted heredoc, set the root password, mask `nftables.service` and `ufw.service`, and enable `foomuuri.service` and `raw-node-firstboot.service`.

The template matches the Ubuntu 26.04 one except for the root password command, which is written as a single-quoted string rather than a list so that it does not depend on newer Subiquity behaviour. Both forms set the same password.

## Notes on the autoinstall file

A `late-commands` entry written as a string runs through `sh -c`, with the new system mounted at `/target` <a id="cite-canonical-autoinstall-reference"></a>([Canonical, n.d.-a](#ref-canonical-autoinstall-reference)). The file-writing entries use YAML block scalars, so the heredocs reach the shell with their indentation stripped and the files come out identical to the canonical versions. The root password hash is wrapped in single quotes so the shell does not expand its `$` characters. A crypt hash never contains a single quote, so this is safe.

The same reference explains the rest of the file:

- Users in `identity` are created during installation, while users in `user-data` are created on first boot. Setting the root password in `late-commands` keeps it with the rest of the install.
- `identity` is not given a `groups` key. The default groups already include `sudo`, and a plain list would replace them.
- `allow-pw` defaults to `false` when `authorized-keys` is present. It is set to `false` explicitly anyway, so the template states key-only SSH (contract RN-SSH-03) instead of relying on a default.
- `authorized-keys` installs the key for the initial user, so no manual `authorized_keys` handling is needed.

`hostname` exists only inside `identity`, and `identity` rejects any key it does not define <a id="cite-canonical-autoinstall-schema"></a>([Canonical, n.d.-b](#ref-canonical-autoinstall-schema)). Do not add a top-level `hostname`.

The installer writes `/etc/ssh/sshd_config.d/50-cloud-init.conf` with its own `PasswordAuthentication` line <a id="cite-launchpad-2088207"></a>([Launchpad, 2024](#ref-launchpad-2088207)). Because Ubuntu includes `sshd_config.d/*.conf` at the top of `sshd_config` and `10-local-auth.conf` sorts first, the canonical drop-in wins, as RN-07 requires. Subiquity 24.04.5 passes `allow-pw` to cloud-init as `ssh_pwauth`, and cloud-init's `set_passwords` module turns `false` into `PasswordAuthentication no` in that file, so both files say `no`. Run against the raw node drop-ins with a `50-cloud-init.conf` that says `yes`, the `sshd -G` of `openssh-server` 1:9.6p1-3ubuntu13.20 still reports `passwordauthentication no` and `kbdinteractiveauthentication no`.

cloud-init runs its user, SSH and password modules once, in its network stage, `cloud-init.service`, with cloud-init 25.1.4 from the security pocket or 26.2 from updates. Subiquity also has cloud-init write `/etc/cloud/cloud-init.disabled` at the end of the first boot, so no cloud-init module runs at a later boot and none can turn password login back on. In that stage cloud-init also deletes the SSH host keys made at install time and makes new ones. `raw-node-firstboot.service` is ordered after it, so the report lists the new keys.

The initial user keeps its password for the console and for sudo, which asks for it. Subiquity passes the user to cloud-init with `lock_passwd: false` and without a `sudo` key, and without cloud-init's `default` user, so cloud-init writes no `/etc/sudoers.d/90-cloud-init-users` and creates no `ubuntu` user. sudo is sudo 1.9.15p5-3ubuntu5.24.04.4, whose stock `/etc/sudoers` and `/etc/sudoers.d/README` have no `NOPASSWD` or `!authenticate`.

The answer file also writes `/etc/cloud/cloud.cfg.d/99-raw-node.cfg` with `preserve_hostname: true`, so that cloud-init does not reset the suffixed hostname that `raw-node-firstboot` sets.

Since Ubuntu 22.10, OpenSSH starts through `ssh.socket` rather than a permanently running `ssh.service`, and 24.04 follows that default <a id="cite-ubuntu-ssh-socket"></a>([Ubuntu Community Hub, n.d.](#ref-ubuntu-ssh-socket)). The file therefore does not enable `ssh.service`. With socket activation the port belongs to `ssh.socket`, not to sshd. `sshd-socket-generator`, which the `openssh-server` package installs, reads the effective sshd configuration at every boot and writes the socket's listen addresses from it. Run against a `Port` line in `sshd_config.d`, the packaged generator produced a socket for that port, so `20-raw-node-port.conf` is enough and no socket override is written. For port 22 the generator writes nothing and the socket's own default applies.

The firewall is Foomuuri 0.22 from the `universe` component, so the installer needs network access to the archive. This version has no `foomuuri-boot.service`, so `foomuuri.service` alone loads the rules, before `network-pre.target`. `ufw` is installed on Ubuntu Server with its service enabled, although it loads no rules. `late-commands` mask `ufw.service` and `nftables.service`, so that Foomuuri is the only firewall manager, as RN-08 requires, and nothing can flush its table.

The answer file also writes `/etc/systemd/system/foomuuri.service.d/10-raw-node.conf`, which turns off the default dependencies of `foomuuri.service`. Without it the unit is ordered after `sysinit.target`, while cloud-init's network stage on 24.04 is ordered before `sysinit.target` and after the network is online. The result is an ordering cycle, which systemd breaks by not starting one of the units. Checked against the packaged unit files, the unit it dropped was `cloud-init.service`, the stage that installs the SSH key at first boot. See [contracts.md](contracts.md#notes-on-the-firewall-clauses).

`python3` is part of Ubuntu Server already. It is listed in `packages` so that RN-09 is explicit in the answer file.

The Subiquity documentation cited here is the current version. Subiquity can refresh itself at the start of an installation when the machine has network access, so the version that actually runs may be newer than the one on the 24.04.5 ISO. Every key used above has been part of autoinstall since well before 24.04.

## Requirement coverage

| ID | How this answer file meets it |
| --- | --- |
| RN-01 | `identity.hostname` set to `ub2404-vt`; suffix added by `raw-node-firstboot`; kept by `preserve_hostname` |
| RN-02 | `locale`, `keyboard`, `timezone`; network time through `systemd-timesyncd`, enabled by default |
| RN-03 | `usermod --password` in `late-commands`; root SSH login refused by the drop-in |
| RN-04 | `identity.username` and `identity.password`, the hash from the Ansible controller; default groups include `sudo`; no sudoers rule written |
| RN-05 | `ssh.authorized-keys` |
| RN-06 | `ssh.install-server`; listening through `ssh.socket`, on the port `sshd-socket-generator` reads from `20-raw-node-port.conf` |
| RN-07 | Heredocs write `/etc/ssh/sshd_config.d/10-local-auth.conf`, which sorts before `50-cloud-init.conf`, and `20-raw-node-port.conf` |
| RN-08 | `foomuuri` in `packages`; heredocs write `/etc/foomuuri/00-raw-node.conf` and the `foomuuri.service` drop-in; `systemctl mask nftables.service ufw.service`; `systemctl enable foomuuri.service` |
| RN-09 | `python3` in `packages` |
| RN-10 | `interactive-sections: [storage]` |
| RN-11 | Only crypt hashes and the public key appear in the file |
| RN-12 | Heredocs write the report scripts, unit and `raw-node.conf`; `systemctl enable raw-node-firstboot.service` |

## Ubuntu-specific checks

Run the baseline [acceptance checks](raw-node-baseline.md#acceptance-checks), then:

```sh
lsb_release -d
systemctl status ssh.socket
systemctl cat ssh.socket
systemctl is-enabled ufw nftables
systemctl status foomuuri
systemctl cat foomuuri
cloud-init status --long
journalctl -b | grep -i -E 'ordering cycle|deleted to break'
ls /etc/ssh/sshd_config.d/
cat /etc/ssh/sshd_config.d/50-cloud-init.conf
ls /etc/sudoers.d/ /etc/cloud/cloud-init.disabled
id ubuntu
hostnamectl --static
```

`lsb_release -d` should report Ubuntu 24.04. `ssh.socket` should be active and enabled. `ssh.service` may show as inactive until the first connection arrives, which is expected with socket activation. When `ssh_port` is not 22, `systemctl cat ssh.socket` should show a generated drop-in with that port. `ufw` and `nftables` should both be `masked`. `foomuuri.service` should be active, and `systemctl cat foomuuri` should show the `10-raw-node.conf` drop-in. `cloud-init status` should report `done` without errors, and the journal search should print nothing. `10-local-auth.conf` should be listed before `20-raw-node-port.conf` and `50-cloud-init.conf`, and `50-cloud-init.conf` should say `PasswordAuthentication no`. `/etc/sudoers.d/` should hold only `README`, `/etc/cloud/cloud-init.disabled` should exist, and `id ubuntu` should report no such user. The hostname should be `ub2404-vt-` followed by four hexadecimal digits, and should survive a reboot.

## Use as a Linux Mint 22 base

Linux Mint 22 is built on the Ubuntu 24.04 archive. Point releases only refresh the installation media, so once the node runs `apt update` it uses the same `noble` archive that Mint 22 is built on. A raw node installed from this document can therefore serve as the base for a Mint 22 desktop.

The result is not a supported Mint installation, and a later move to Mint 23, which is based on Ubuntu 26.04, should be planned as a rebuild rather than an upgrade. The desktop layer belongs in Ansible, and it needs these adjustments:

- Switch netplan to `renderer: NetworkManager`, otherwise the desktop shows the network as unmanaged.
- Keep Mint's `ufw` inactive or remove it, so that Foomuuri remains the only firewall manager.
- Expect apt to propose removing `snapd`, because Mint blocks it. Review the list before confirming.
- Install `foomuuri-firewalld` on the desktop, unlike on servers, so that NetworkManager can switch zones.
- cloud-init stays installed from the server install. It is harmless, and it can be turned off with `touch /etc/cloud/cloud-init.disabled` once the desktop is built.

## Review history

| Item | Status | Detail |
| --- | --- | --- |
| Answer file derived from the combined document | 0.1.1 | Same accounts, SSH policy and `nftables` ruleset as the Ubuntu 26.04 file. |
| Root password command | 0.1.1 | Single-quoted string instead of the list form. |
| Installation media | 0.1.2 | Moved from 24.04.4 to 24.04.5. |
| `python3` | 0.2.0 | Requested explicitly for RN-09. |
| Hostname | 0.3.0 | `ub2404-vt` plus a MAC suffix at first boot, replacing `ubuntu-server`. |
| First-boot report and `preserve_hostname` | 0.3.0 | RN-12. |
| Firewall | 0.3.0 | Foomuuri with `/etc/foomuuri/00-raw-node.conf` instead of `/etc/nftables.conf`. `nftables.service` and `ufw.service` masked. Drop-in for `foomuuri.service`. |
| SSH port | 0.3.0 | `20-raw-node-port.conf`, from the `ssh_port` input. |
| SSH password login | 0.4.0 | `allow-pw: false` and `PasswordAuthentication no`, for contract RN-SSH-03. |

## Qualification

This is a reviewed configuration template, not yet a tested turnkey installer. Validate it in an Incus virtual machine with the 24.04.5 ISO before using it on physical hardware.

## Change log

### 0.4.0, 2026-11-17

- Key-only SSH: `allow-pw: false`, and `PasswordAuthentication no` and `PermitEmptyPasswords no` in the policy drop-in.
- `identity.password` takes the batch hash from the Ansible controller, and `raw-node.conf` records the batch ID when `password_batch` is set.
- The confirmation follows the raw node contract 0.3.0. See [contracts.md](contracts.md).

### 0.3.0, 2026-11-15

- Firewall changed to Foomuuri 0.22: `foomuuri` added to the packages, `/etc/foomuuri/00-raw-node.conf` written instead of `/etc/nftables.conf`, `nftables.service` and `ufw.service` masked, `foomuuri.service` enabled.
- A drop-in for `foomuuri.service` keeps it out of an ordering cycle with cloud-init.
- SSH port drop-in written from the `ssh_port` input, followed by `ssh.socket` through `sshd-socket-generator`.
- The confirmation follows the raw node contract 0.1.0. See [contracts.md](contracts.md).

### 0.3.0, 2026-11-14

- The answer file is now rendered by `bin/build` from [`ubuntu/24.04/user-data.j2`](../../ubuntu/24.04/user-data.j2) and is linked rather than copied here.
- New stick paths: `/iso/ubuntu/24.04/` and `/answer-files/ubuntu/24.04/user-data`.
- Hostname `ub2404-vt` with a first-boot MAC suffix, kept by a cloud-init `preserve_hostname` setting; added RN-12 coverage.
- Mint 22 base section: added the point-release reasoning and the cloud-init note from the session notes.
- Renamed from `ventoy-automated-configuration-for-ubuntu-24.04-server-v0.2.0.md`; versions are tracked in the front matter.

### Earlier versions

- 0.2.0: distro-agnostic content moved to the baseline, `python3` added, requirement coverage table added.
- 0.1.2: moved to the 24.04.5 installation media and added the handover to Ansible.
- 0.1.1: first release, derived from the Ubuntu section of the combined document.

## References

<a id="ref-canonical-autoinstall-reference"></a>
Canonical. (n.d.-a). *Autoinstall configuration reference manual*. Subiquity documentation. Retrieved November 14, 2026, from https://canonical-subiquity.readthedocs-hosted.com/en/latest/reference/autoinstall-reference.html
[Return to citation](#cite-canonical-autoinstall-reference)

<a id="ref-canonical-autoinstall-schema"></a>
Canonical. (n.d.-b). *Autoinstall schema*. Subiquity documentation. Retrieved November 14, 2026, from https://canonical-subiquity.readthedocs-hosted.com/en/latest/reference/autoinstall-schema.html
[Return to citation](#cite-canonical-autoinstall-schema)

<a id="ref-gupta-2026"></a>
Gupta, U. (2026, September 10). *Ubuntu 24.04.5 LTS released* [Online forum post]. Ubuntu Community Hub. https://discourse.ubuntu.com/t/ubuntu-24-04-5-lts-released/87608
[Return to citation](#cite-gupta-2026)

<a id="ref-jacquet-2026"></a>
Jacquet, F. (2026, February 12). *Ubuntu 24.04.4 LTS released* [Mailing list post]. ubuntu-announce. https://lists.ubuntu.com/archives/ubuntu-announce/2026-February/000321.html
[Return to citation](#cite-jacquet-2026)

<a id="ref-larabel-2026"></a>
Larabel, M. (2026, September 10). *Ubuntu 24.04.5 LTS released with Linux 7.0 HWE stack*. Phoronix. https://phoronix.com/news/Ubuntu-24.04.5-LTS
[Return to citation](#cite-larabel-2026)

<a id="ref-launchpad-2088207"></a>
Launchpad. (2024). *Bug #2088207: cloud-init enables ssh password auth in an unexpected config file*. https://bugs.launchpad.net/bugs/2088207
[Return to citation](#cite-launchpad-2088207)

<a id="ref-linuxiac-incus-7"></a>
Linuxiac. (2026). *Incus 7.0 LTS container & virtual machine manager released*. https://linuxiac.com/incus-7-0-lts-container-and-virtual-machine-manager-released/
[Return to citation](#cite-linuxiac-incus-7)

<a id="ref-ubuntu-ssh-socket"></a>
Ubuntu Community Hub. (n.d.). *sshd now uses socket-based activation (Ubuntu 22.10 and later)*. Retrieved November 14, 2026, from https://discourse.ubuntu.com/t/sshd-now-uses-socket-based-activation-ubuntu-22-10-and-later/30189
[Return to citation](#cite-ubuntu-ssh-socket)
