---
name: uc-raw-node-contract
org: universal-cake
lang: en
type: contract
version: 0.1.0
date: 2026-11-15
---

# Raw node contract

Version: 0.1.0

Status: Draft

Author: Christopher Steel

This contract defines the state of a raw node at the moment a producer hands it over, and what Ansible ingress may rely on when it receives the node. It is the agreement between raw node producers (egress) and the Ansible controller (ingress). It does not describe how a producer fulfils it, or what happens after ingress.

## Stages

| Stage | What it does | Changes the node | Covered here |
| --- | --- | --- | --- |
| Egress | A producer builds a raw node | Yes | Yes, as obligations |
| Ingress | Discovery finds the node and checks it against this contract | No, read-only | Yes, as what it may rely on |
| Onboarding | Prepares the node for normalization, and handles nodes that failed the contract | Yes | No |
| Normalization | Brings the node to its managed state | Yes | No |

## Scope

Every clause below describes the raw node as it is when the producer hands it over. Ingress may rely on every clause and may check any of them without changing the node. The contract does not say how a producer meets a clause, and it does not say what happens when a clause fails.

The contract is the same for every Linux distribution. Which producers and targets implement it is recorded outside this document, by each producer.

## Shared parameters

Both sides know these values for each node:

| Parameter | Meaning |
| --- | --- |
| `initial_username` | The temporary account used for ingress |
| `ssh_public_key` | The public key whose private key ingress holds |
| `ssh_port` | The SSH port. Default 22. |
| `ssh_sources` | The addresses or networks allowed to reach SSH. Empty means any. |

Password hashes and other secrets used to build the node belong to the producer. Ingress never needs them.

## Clauses

### Network

| ID | The raw node provides |
| --- | --- |
| RN-NET-01 | A hostname that is set and is not the installer's default |

### Accounts

| ID | The raw node provides |
| --- | --- |
| RN-ACC-01 | The account `initial_username` exists, and its home directory holds only `.ssh/authorized_keys` and the distribution's defaults for a new home |
| RN-ACC-02 | `initial_username` can log in over SSH with `ssh_public_key` |
| RN-ACC-03 | `initial_username` can log in over SSH with its password, as a recovery path |
| RN-ACC-04 | `initial_username` can obtain root privileges through sudo, with its password |
| RN-ACC-05 | The `root` account has a password and cannot log in over SSH |

### SSH

| ID | The raw node provides |
| --- | --- |
| RN-SSH-01 | The SSH server listens on `ssh_port` |
| RN-SSH-02 | SSH is reachable from `ssh_sources`, or from anywhere when the list is empty, on at least one address reported in `interfaces` |

### Firewall

These clauses are stated in nftables terms and are meant to outlast any one firewall manager.

| ID | The raw node provides |
| --- | --- |
| RN-FW-01 | nftables is the only packet filter: there are no iptables-legacy rules, and no other firewall manager is active |
| RN-FW-02 | One firewall manager owns one nftables table, and nothing flushes the whole ruleset at boot or when a service stops |
| RN-FW-03 | Inbound, only SSH as in RN-SSH-02, DHCP and DHCPv6 replies, ping, and the ICMP and ICMPv6 that networking needs are accepted. Outbound, everything is accepted. Forwarded, nothing is accepted. |
| RN-FW-04 | The firewall is in force before the network comes up at boot |

### Firewall manager

This clause names the current manager. Replacing it is a major change to the contract, while the firewall clauses above stay.

| ID | The raw node provides |
| --- | --- |
| RN-FWM-01 | The firewall manager is Foomuuri, and its configuration is only the file `/etc/foomuuri/00-raw-node.conf` |

### Report

| ID | The raw node provides |
| --- | --- |
| RN-REP-01 | The file `/var/lib/raw-node/confirmation.yml` exists, is readable by `initial_username`, and holds the fields defined below, measured on the node |

Identifiers are never reused. A removed clause keeps its identifier retired, so a report naming a clause always means the same thing.

## Confirmation

### Format

The confirmation is YAML, keyed by the node's hostname:

```yaml
deb13-vt-a1b2:
  contract: 0.1.0
  baseline: 0.3.0
  timestamp: 2026-11-15T19:35:00-05:00
  machine_id: 4c1f0a9e2b7d4e8f9a0b1c2d3e4f5a6b
  user: initial
  sudo: true
  ssh_port: 22
  firewall: foomuuri
  firewall_active: true
  python: /usr/bin/python3
  interfaces:
    - name: enp0s31f6
      type: ethernet
      mac: 8c:8c:aa:11:22:33
      state: up
      default_route4: true
      default_route6: true
      ip: [192.0.2.10]
      ip6: [2001:db8::10]
    - name: wlp0s20f3
      type: wireless
      mac: 70:cf:49:44:55:66
      state: down
      default_route4: false
      default_route6: false
      ip: []
      ip6: []
```

### Fields

Every value is measured on the node when the report runs, not copied from the producer's input.

| Field | Meaning |
| --- | --- |
| Top-level key | The node's hostname |
| `contract` | The version of this contract the node implements |
| `baseline` | The producer's own baseline version |
| `timestamp` | When the report ran, in ISO 8601 with offset |
| `machine_id` | The contents of `/etc/machine-id`. Identifies this installation, and changes on reinstall. |
| `user` | `initial_username` if that account exists, otherwise empty |
| `sudo` | Whether `initial_username` can use sudo |
| `ssh_port` | The port the SSH server listens on |
| `firewall` | The name of the active firewall manager |
| `firewall_active` | Whether the firewall manager is running |
| `python` | The path of the system Python 3 interpreter, or empty when there is none. Reported, not promised. |
| `interfaces` | Every network interface except loopback, as found |
| `interfaces[].name` | The kernel name of the interface |
| `interfaces[].type` | `ethernet`, `wireless` or `other` |
| `interfaces[].mac` | The permanent hardware address, not one that may be randomised |
| `interfaces[].state` | `up` or `down` |
| `interfaces[].default_route4` | Whether the interface holds an IPv4 default route |
| `interfaces[].default_route6` | Whether the interface holds an IPv6 default route |
| `interfaces[].ip` | The IPv4 addresses on the interface, an empty list when there are none |
| `interfaces[].ip6` | The global IPv6 addresses on the interface, an empty list when there are none |

The network interface fields are facts only. The contract makes no promise about how many interfaces a node has, which of them are up, or which holds a default route.

### Identity

`machine_id` and the interfaces' MAC addresses together let ingress match a node with what it already knows:

| `machine_id` | MAC addresses | Reading |
| --- | --- | --- |
| Known | Known | The same installation on the same hardware |
| Known | New | The same installation, on other hardware or with other network devices |
| New | Known | The same hardware, reinstalled |
| New | New | A new node |

Anyone with root on a node can change both values. Ingress may use them to match and track nodes, but never to decide whether to trust one.

## Not guaranteed

The contract says nothing about these, and ingress must not rely on them:

- package versions, update state, and whether automatic updates run
- accounts other than `initial_username` and `root`, and services other than SSH and the firewall
- disk layout
- locale, time zone and the clock
- Python
- anything about network interfaces beyond what the confirmation reports

## Versioning

The contract has its own version, independent of any producer's baseline. Every node records the version it implements in the confirmation's `contract` field.

| Change | Version part |
| --- | --- |
| A clause is removed, weakened or changed in meaning, a confirmation field is removed, renamed or changes meaning, or the firewall manager in RN-FWM-01 changes | Major |
| A clause or a confirmation field is added | Minor |
| Wording is clarified without changing what either side may rely on | Patch |

While the major version is 0, a minor version may also be incompatible, and the changelog says when it is.

## Changelog

| Version | Status | Notes |
| --- | --- | --- |
| 0.1.0 | Draft | First version, agreed in discussion: clauses for hostname, accounts, SSH, the nftables firewall and its manager, and the confirmation report with interfaces and identity |
