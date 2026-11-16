---
name: raw-node-interface
org: universal-cake
lang: en
type: specification
version: 0.1.0
date: 2026-11-15
---

# Raw node interface

Version: 0.1.0

Status: Draft

Author: Christopher Steel

This document defines what a raw node is: the input that describes one, what every raw node provides once installed, what it reports back, and what Ansible may rely on when it takes over. It describes raw node baseline 0.3.0, as produced by `uc-ventoy`.

## Summary

A raw node is a freshly installed host with the smallest set of features needed for Ansible to take over safely: one administrative user with an SSH key and sudo, an SSH server, Python, and a minimal Foomuuri firewall that allows SSH and little else. Nothing else is configured.

The interface has three parts:

- **Input.** One file, `raw-node-input.yml`, holds every value that differs between installations.
- **Node.** Every raw node, whatever produced it, provides the same accounts, services, firewall and files.
- **Output.** At first boot every raw node writes `/var/lib/raw-node/confirmation.yml`, measured on the node, from which Ansible builds its inventory.

Producers can change, for example a Ventoy preseed today and an Incus or VPS image later. The interface stays the same, and its version is the baseline version recorded on every node.

## Terms

| Term | Meaning |
| --- | --- |
| Raw node | A host installed to this interface and not yet configured by Ansible |
| Producer | Whatever installs a raw node. Today: `bin/build` in `uc-ventoy`, rendering installer answer files for a Ventoy stick. |
| Target | One distribution and version a producer supports, such as `debian/13` |
| Baseline version | The version of this interface a node was built to, recorded as `RAW_NODE_BASELINE` on the node |
| Handoff | The first Ansible run against a raw node, after which it is no longer raw |

## Lifecycle

1. The operator fills in `raw-node-input.yml` on the workstation.
2. `bin/build` validates the input, renders one answer file per target, self-tests each, and optionally writes them to the Ventoy stick.
3. The installer runs from the stick. Partitioning stops for manual confirmation, everything else is automatic.
4. At first boot, `raw-node-firstboot.service` adds a MAC suffix to the hostname, writes the confirmation, and shows it at the console login prompt.
5. The operator copies the confirmation into the Ansible inventory, and the handoff begins.

## Input

### Keys

The input file is `raw-node-input.yml` at the root of `uc-ventoy`. Unknown keys are reported as warnings and ignored.

| Key | Required | Type | Default | Validation |
| --- | --- | --- | --- | --- |
| `root_password_hash` | Yes | String | None | A crypt hash, such as `$6$...` or `$y$...` |
| `initial_username` | Yes | String | None | A valid Linux user name: lowercase, starts with a letter or underscore, at most 32 characters |
| `initial_password_hash` | Yes | String | None | A crypt hash |
| `ssh_public_key` | Yes | String | None | One OpenSSH public key line: ed25519, RSA, ECDSA or a security key type |
| `ssh_port` | No | Integer | `22` | From 1 to 65535 |
| `ssh_sources` | No | List of strings | Empty, meaning anywhere | Each entry an IPv4 or IPv6 address or network. Host bits are refused, so `192.0.2.1/24` is an error. A bare address becomes a /32 or /128. |

### Character rules

Required values must not contain a single quote, double quote, backslash or line break, because values are embedded in shell commands inside installer answer files. A value containing `DUMMY` is refused unless the build is run with `--allow-dummy`, which is for test builds only.

### Example

```yaml
root_password_hash: $6$...
initial_username: initial
initial_password_hash: $6$...
ssh_public_key: ssh-ed25519 AAAA... initial@laptop
ssh_port: 22
ssh_sources:
  - 192.0.2.0/24
  - 2001:db8::/32
```

## What every raw node provides

### Accounts

| Account | Password | SSH | Privileges |
| --- | --- | --- | --- |
| `root` | From `root_password_hash` | Login refused | Console only |
| `initial_username` | From `initial_password_hash` | Key from `ssh_public_key`, or password | Member of `sudo`. sudo asks for the password. |

### SSH

The SSH server is installed and enabled. Two drop-in files set its behaviour:

| File | Settings |
| --- | --- |
| `/etc/ssh/sshd_config.d/10-local-auth.conf` | `PermitRootLogin no`, `PubkeyAuthentication yes`, `PasswordAuthentication yes`, `KbdInteractiveAuthentication no`, `UsePAM yes` |
| `/etc/ssh/sshd_config.d/20-raw-node-port.conf` | `Port` set to `ssh_port`, written even when it is 22 |

### Firewall

Every raw node runs Foomuuri, which manages one nftables table, `inet foomuuri`. The configuration is one file, `/etc/foomuuri/00-raw-node.conf`, rendered once by `bin/build` from `shared/foomuuri/00-raw-node.conf.j2`, so every target ships identical rules:

| Direction | Allowed |
| --- | --- |
| In, from the `public` zone | DHCP and DHCPv6 replies, ping, SSH on `ssh_port` from `ssh_sources` (or from anywhere when the list is empty) |
| Out, to the `public` zone | Everything |
| Forwarded | Nothing |

Anything not allowed is dropped and logged, as Foomuuri does by default.

The `public` zone holds interfaces named `en*` (wired), `eth*` (containers and most VPS) and `wl*` (wireless). Foomuuri's catch-all interface `*` would be simpler, but it only arrived in Foomuuri 0.33 <a id="cite-foobaroy-2026"></a>[(FoobarOy, 2026)](#ref-foobaroy-2026), and the versions shipped by Debian 13 (0.27) and Ubuntu 24.04 (0.22) generate rules that nftables rejects when it is used. An interface outside these patterns, such as a bridge, is not in any zone and receives no SSH.

Services:

- `foomuuri.service` is enabled. `foomuuri-boot.service` is enabled where it exists (Foomuuri 0.24 and later), and loads the last good ruleset early in boot.
- `nftables.service` is masked. Its stop action on Debian runs `nft flush ruleset`, which would remove Foomuuri's table along with every other.

The file carries no `ansible-role-foomuuri` marker, so the role's default purge removes it on the handoff run, in the same reload that installs the role's own rules.

### Packages

The standard system utilities and an SSH server, plus `sudo`, `nftables`, `python3` and `foomuuri`. Packages are not upgraded during installation, and automatic updates are not configured.

### System settings

| Setting | Value |
| --- | --- |
| Locale | `en_CA.UTF-8` |
| Keyboard | `us` |
| Time zone | `America/Toronto`, hardware clock in UTC, NTP on |
| Domain | `home.arpa` |
| Hostname | `<prefix><version>-vt-<suffix>`, for example `deb13-vt-a1b2`. The prefix comes from the distribution (`deb`, `ub`, `fed`, `rocky`), and the suffix is the last four hex digits of the MAC address of the interface holding the default route, added at first boot. |

### Files

| Path | Purpose |
| --- | --- |
| `/etc/raw-node/raw-node.conf` | Shell variables describing how the node was built, read by `raw-node-firstboot` |
| `/usr/local/sbin/raw-node-report` | Prints the confirmation, measured on the node |
| `/usr/local/sbin/raw-node-firstboot` | Runs once at first boot: hostname suffix, confirmation, console notice |
| `/etc/systemd/system/raw-node-firstboot.service` | Starts the above once, skipped while the confirmation exists |
| `/etc/issue.d/50-raw-node.issue` | Shows the confirmation at the console login prompt |
| `/var/lib/raw-node/confirmation.yml` | The output, see below |

`/etc/raw-node/raw-node.conf` holds:

| Variable | Example | Meaning |
| --- | --- | --- |
| `RAW_NODE_USER` | `initial` | The administrative user |
| `RAW_NODE_TARGET` | `debian/13` | The target the node was built for |
| `RAW_NODE_BASELINE` | `0.3.0` | The interface version |
| `RAW_NODE_FIREWALL` | `foomuuri` | The firewall shipped |
| `RAW_NODE_SSH_PORT` | `22` | The SSH port |
| `RAW_NODE_SSH_SOURCES` | `"192.0.2.0/24 2001:db8::/32"` | The SSH sources, space separated, empty for anywhere |

## Output

At first boot, once the network is online, the node writes `/var/lib/raw-node/confirmation.yml`. Every value is measured on the node, not copied from the input.

```yaml
deb13-vt-a1b2:
  timestamp: 2026-11-15T16:20:00-05:00
  ip: 192.0.2.10
  ip6: 2001:db8::10
  user: initial
  sudo: true
```

| Field | Meaning |
| --- | --- |
| Top-level key | The hostname, after the MAC suffix was added |
| `timestamp` | When the report ran, ISO 8601 with offset |
| `ip`, `ip6` | The source addresses the kernel would use to reach the outside, empty if none |
| `user` | The administrative user, empty if it does not exist |
| `sudo` | Whether that user can use sudo |

The report exits with status 0 only when the user exists and has sudo, and `raw-node-firstboot.service` fails otherwise, so `systemctl --failed` shows a node that did not come up correctly. Deleting `confirmation.yml` makes the service run again at the next boot.

## Handoff to Ansible

### What Ansible may rely on

- SSH on `ssh_port`, reachable from `ssh_sources`, as `initial_username` with the key.
- sudo for that user, with the user's password as the become password.
- `/usr/bin/python3`.
- Foomuuri installed and active, with configuration only in `/etc/foomuuri/00-raw-node.conf`.
- The values in `/etc/raw-node/raw-node.conf`.

### What the handoff run must do

- **Keep SSH open throughout.** The firewall it installs must allow SSH from the same sources on the same port. Inventory should read `ssh_port` and `ssh_sources` from the same `raw-node-input.yml` the node was built from, so the two cannot drift.
- **Replace the raw node firewall.** `ansible-role-foomuuri` with its default `foomuuri_purge: unmanaged` removes `00-raw-node.conf` and installs its base fragment in one checked reload.
- **Update packages.** Raw nodes are installed without upgrades and without automatic updates.
- **Harden SSH.** Password authentication is on for raw nodes, as a recovery path. The handoff should turn it off once key access is confirmed.

After the handoff the node is no longer raw, and nothing in this interface applies to it.

## Versioning

The interface version and the baseline version are the same number, set in `bin/build` as `BASELINE_VERSION` and recorded on every node.

| Change | Version part |
| --- | --- |
| Something Ansible relied on is removed or changed: an account, the SSH or sudo behaviour, the firewall manager, the confirmation format | Major |
| Something is added that does not break earlier handoffs, such as a new optional input key or a new field in `raw-node.conf` | Minor |
| A fix that leaves the interface as documented | Patch |

While the major version is 0, as now, a minor version may also break compatibility, and the history below says when it does.

| Baseline | Changes |
| --- | --- |
| 0.2.0 | Firewall is a static `/etc/nftables.conf` loaded by `nftables.service`, SSH on port 22 from anywhere |
| 0.3.0 | Firewall is Foomuuri with `/etc/foomuuri/00-raw-node.conf`. `nftables.service` masked. New optional input keys `ssh_port` and `ssh_sources`. New SSH drop-in for the port. New `RAW_NODE_FIREWALL`, `RAW_NODE_SSH_PORT` and `RAW_NODE_SSH_SOURCES` in `raw-node.conf`. Not compatible with 0.2.0 for firewall handling. |

Nodes on 0.2.0 still run `nftables.service`, which flushes the whole ruleset when stopped. Ansible must handle them as hosts with a foreign firewall, not as 0.3.0 raw nodes.

## Producers and targets

| Producer | Target | Answer file | Baseline | Status |
| --- | --- | --- | --- | --- |
| `uc-ventoy` | `debian/13` | `preseed.cfg` | 0.3.0 | Template updated, not yet installed on a host |
| `uc-ventoy` | Ubuntu (autoinstall) | `user-data` | Reports 0.3.0, ships 0.2.0 firewall | Template not yet updated, see open questions |

`BASELINE_VERSION` is global in `bin/build`, so every target reports the same baseline. A target must not be built under a baseline it does not implement.

## Security notes

- The built answer files contain both password hashes, so the Ventoy stick needs the same care as key material. `bin/build` writes them with mode `0600`.
- Raw nodes allow SSH password authentication. Setting `ssh_sources` is the cheapest way to limit who can try it before the handoff.
- If root and the initial user share a password hash, they share a password.
- If the Foomuuri file were invalid, Foomuuri would fail to start at first boot with no earlier good ruleset to fall back on, and the node would have no firewall. `bin/build` checks the file with `foomuuri check` when Foomuuri is installed on the workstation. The check needs root: `sudo bin/build --check-firewall-only`.

## Open questions

| Question | Options |
| --- | --- |
| Ubuntu autoinstall template | Update it to 0.3.0 now (Foomuuri 0.22 from `universe`, `foomuuri.service` only), or give targets their own baseline version so Ubuntu can stay on 0.2.0 until it is updated |
| Incus and VPS producers | A cloud-init or Incus profile producer implementing the same interface, so test nodes and production servers start from the same baseline |
| Measured firewall state in the confirmation | Add fields such as `firewall` and `ssh_port`, measured on the node, so Ansible can verify the baseline before the handoff |
| Become password | Keep sudo with a password, or allow `NOPASSWD` for the initial user until the handoff |
| Non-matching interface names | Whether the `public` zone patterns need to cover other names, such as `bond*` or `br*`, on future targets |

## References

<a id="ref-foobaroy-2026"></a>
[Return to citation](#cite-foobaroy-2026)
FoobarOy. (2026). *Changelog* [Foomuuri 0.33 and 0.34 entries]. GitHub. https://github.com/FoobarOy/foomuuri/blob/main/CHANGELOG.md

## Changelog

| Version | Status | Notes |
| --- | --- | --- |
| 0.1.0 | Draft | First definition of the raw node interface, describing baseline 0.3.0 |
