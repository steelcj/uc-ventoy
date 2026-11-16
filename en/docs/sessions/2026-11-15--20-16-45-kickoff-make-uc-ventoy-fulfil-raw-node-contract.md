# Kickoff: make uc-ventoy fulfil raw node contract 0.1.0

## Goal

Make the uc-ventoy targets debian/13, ubuntu/24.04 and ubuntu/26.04 produce raw nodes that fulfil the raw node contract, version 0.1.0. uc-ventoy is the contract's egress (producer).

## Read first

1. The contract: uc-contracts, en/docs/raw-node/uc-raw-node-contract.md. It is authoritative. Do not change it. If a clause cannot be met, or seems wrong, stop and tell me instead of working around it. https://github.com/steelcj/uc-contracts
2. bin/build, every *.j2 template, everything under shared/, raw-node-input.example.yml, and the existing docs under en/docs/.
3. uc-ventoy-foomuuri.tar.gz, if present: earlier draft changes (Foomuuri firewall, ssh_port and ssh_sources inputs, SSH port drop-in). Treat it as a starting point to review, not as correct. It was written without access to this repository.

## Resources

This files contains an update created using Claude Desktop before the contract was finalize. May or may not be helpful

```bash
uc-ventoy-foomuuri.tar.gz
```



## Required changes

### Firewall (RN-FW-01 to RN-FW-04, RN-FWM-01)

- Replace shared/nftables/nftables.conf with one Foomuuri file, /etc/foomuuri/00-raw-node.conf, rendered once by bin/build from a shared template and embedded identically in every target.
- Zone interfaces: use the patterns en* eth* wl*, not the catch-all *. The catch-all needs Foomuuri 0.33 or later, and the targets ship 0.27 (Debian 13), 0.22 (Ubuntu 24.04) and 0.31 (Ubuntu 26.04).
- Mask nftables.service on all targets, because its stop action flushes the whole ruleset. Also mask ufw.service on Ubuntu.
- Enable foomuuri.service everywhere, and foomuuri-boot.service on debian/13 and ubuntu/26.04 only. 0.22 has no foomuuri-boot.
- RN-FW-04 requires the firewall to be in force before the network comes up. Check the unit ordering of each target's Foomuuri version, especially 24.04 without foomuuri-boot, and report what you find.
- Add a foomuuri check of the rendered file to bin/build (it needs root; warn without it). Remove the nft check of nftables.conf once nothing uses it.

### Inputs

- Optional ssh_port (default 22) and ssh_sources (list of addresses or networks, default empty, meaning any), validated in bin/build and documented in raw-node-input.example.yml.
- The SSH rule in the Foomuuri file and an sshd Port drop-in both come from these values.

### Confirmation report (RN-REP-01)

Rewrite shared/bin/raw-node-report so its output matches the contract's confirmation format and field table exactly:

- New fields: contract, baseline, machine_id, ssh_port, firewall, firewall_active, python, and the interfaces list.
- ssh_port is measured from the running sshd configuration, not copied from input.
- python is the Python 3 path, or empty when there is none.
- interfaces lists every interface except loopback: type (ethernet, wireless or other), the permanent MAC (not a randomised one), state, default_route4, default_route6, and ip and ip6 as lists. ip6 holds global addresses only.

Constraints:

- POSIX sh, shellcheck clean.
- No single quotes and no backslashes, because the script is embedded in preseed printf arguments.
- Output must be valid YAML.
- Keep the current exit-status meaning.
- Use only tools present on a minimal install of all three targets.

The contract and baseline versions should come from /etc/raw-node/raw-node.conf. Add RAW_NODE_CONTRACT there, set from a CONTRACT_VERSION constant in bin/build. Set BASELINE_VERSION to 0.3.0.

### Clauses to check per target

For each target, confirm from the templates and shared files that these hold, and fix them where they don't:

- RN-NET-01: the hostname is not the installer default after first boot.
- RN-ACC-01: initial's home holds only .ssh/authorized_keys and distribution defaults. Check what Ubuntu's installer and cloud-init add.
- RN-ACC-03: SSH password login works for initial. Check that nothing from cloud-init turns it off.
- RN-ACC-04 and RN-ACC-05: sudo with password for initial; root has a password and SSH root login is refused.
- RN-SSH-01: sshd listens on ssh_port. Ubuntu starts sshd through ssh.socket. Find out whether the socket follows a Port drop-in on 24.04 and 26.04, and if it does not, make it.

### Documentation

- Add en/docs/contracts.md: raw-node, version 0.1.0, role Egress.
- Document which targets implement which contract version.
- Remove or update docs that describe the old nftables firewall.

## Working rules

- Do not commit. Leave all changes for my review.
- Ask before adding dependencies or changing anything outside this repository.
- Record this session in en/docs/ as <timestamp>-session.md, my usual session record.

## Done when

- bin/build passes for all three targets, with self-tests OK and shellcheck clean.
- sudo bin/build --check-firewall-only passes.
- You give me a table with one row per contract clause and one column per target. Each cell says met (and how), not met (and why), or "needs an install to confirm". After it, list the checks I should run on a real installed node of each target.
