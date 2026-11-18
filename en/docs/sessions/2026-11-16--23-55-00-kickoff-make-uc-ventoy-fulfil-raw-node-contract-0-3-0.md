# Kickoff: make uc-ventoy fulfil raw node contract 0.3.0

## Goal

Make the uc-ventoy targets debian/13, ubuntu/24.04 and ubuntu/26.04 produce raw nodes that fulfil the raw node contract, version 0.3.0, as baseline 0.4.0. uc-ventoy is the contract's egress (producer).

## Read first

1. The contract: uc-contracts, `en/docs/node-onboarding-pipeline/1-raw-node-egress/uc-raw-node-contract.md`, version 0.3.0. https://github.com/steelcj/uc-contracts. It is authoritative. Do not change it. If a clause cannot be met, or seems wrong, stop and tell me instead of working around it.
2. In the same repository, for the reasons behind the changes: `en/docs/decisions/uc-privileged-access-decision-v0.3.0.md` and `en/docs/node-policies/privileged-access/uc-privileged-access-policy.md`. The policy applies to normalized nodes, not to raw nodes. Read it for context only.
3. The previous kickoff and session record under `en/docs/sessions/`, `en/docs/contracts.md`, `en/docs/raw-node-baseline.md`, bin/build, every *.j2 template, everything under shared/, and raw-node-input.example.yml.

## What changed in the contract since 0.1.0

| Version | Change | Effect on uc-ventoy |
| --- | --- | --- |
| 0.2.0 | RN-FWM-01 retired: the contract no longer names a firewall manager | Foomuuri stays, as this producer's choice. The firewall clauses RN-FW-01 to RN-FW-04 are unchanged. |
| 0.3.0 | Shared parameter `initial_password` added, held by the controller | The hash in `raw-node-input.yml` now comes from the controller |
| 0.3.0 | RN-ACC-03 retired: no password login over SSH | Password authentication off in sshd |
| 0.3.0 | RN-ACC-06: `initial` can log in at the console with its password | Must keep working |
| 0.3.0 | RN-ACC-07: no sudoers rule contains `NOPASSWD` or `!authenticate` | Check every target, including what cloud-init writes |
| 0.3.0 | RN-ACC-08: `initial_password` is at least 15 characters | Met by the controller's generator |
| 0.3.0 | RN-SSH-03: key-only SSH | sshd and cloud-init settings |
| 0.3.0 | Confirmation fields `sudo_nopasswd`, `ssh_password_auth`, `ssh_host_keys` | New report fields |

## Required changes

### Versions

Set CONTRACT_VERSION to 0.3.0 and BASELINE_VERSION to 0.4.0 in bin/build.

### SSH (RN-SSH-03, RN-ACC-02, RN-ACC-05)

- In shared/ssh/10-local-auth.conf: `PasswordAuthentication no`, `KbdInteractiveAuthentication no`, `PermitEmptyPasswords no`. Keep `PubkeyAuthentication yes` and `PermitRootLogin no`.
- On Ubuntu, set the autoinstall `ssh.allow-pw` to false, so cloud-init writes `PasswordAuthentication no` too. sshd uses the first value it reads and 10-local-auth.conf sorts before 50-cloud-init.conf, but the two must not disagree.
- Check that nothing on any target turns password authentication back on later, such as a cloud-init module that runs at a later boot.

### Console login (RN-ACC-06)

- Confirm that `initial` can log in with its password on the console of each target (getty on tty1, and the serial console if a target enables one). Nothing should have changed here, but the SSH changes must not touch PAM.

### sudo (RN-ACC-04, RN-ACC-07)

- Confirm on each target that no file in /etc/sudoers or /etc/sudoers.d contains `NOPASSWD` or `!authenticate`. On Ubuntu, check what cloud-init writes for the Subiquity identity user, especially /etc/sudoers.d/90-cloud-init-users, and what sudo-rs on 26.04 reads.
- Do not add a sudoers rule for `initial`. Membership of the `sudo` group is how it obtains root.

### Password (RN-ACC-08)

- `initial_password_hash` no longer comes from `mkpasswd` on the workstation. The Ansible controller creates a batch password with `scripts/new_raw_batch.py` in uc-controller, keeps it, and prints its batch ID and a SHA-512 hash. Update raw-node-input.example.yml and the docs to say so. Do not add any way of entering the plain password into uc-ventoy.
- bin/build cannot see the password's length. Record RN-ACC-08 as met by procedure, through the controller's generator.
- Add an optional `password_batch` input, the batch ID, written to /etc/raw-node/raw-node.conf as `RAW_NODE_PASSWORD_BATCH`, so that an operator at the console can tell which batch a node was built with. It is not a contract field and does not go in the confirmation.

### Confirmation report (RN-REP-01)

Extend shared/bin/raw-node-report so its output matches the contract 0.3.0 confirmation format and field table exactly:

- `sudo_nopasswd`: true if any file in /etc/sudoers or /etc/sudoers.d contains `NOPASSWD` or `!authenticate`, otherwise false.
- `ssh_password_auth`: from the running configuration, `sshd -T`. True if `passwordauthentication` or `kbdinteractiveauthentication` is yes, otherwise false.
- `ssh_host_keys`: one entry per /etc/ssh/ssh_host_*_key.pub, with `type` (such as `ssh-ed25519`) and the SHA256 `fingerprint` as printed by `ssh-keygen -l`.
- Quote `mac` and `machine_id` values as YAML strings. They are the same values, but an unquoted MAC of decimal digits, or an all-digit machine ID, is read by YAML 1.1 parsers as a base 60 integer, as the previous session found.
- On Ubuntu, cloud-init generates the host keys at first boot. Make sure the report runs after the keys exist, on every target, and that a missing key fails the report rather than producing an empty list.

The constraints from the previous kickoff still hold: POSIX sh, shellcheck clean, no single quotes and no backslashes, valid YAML, the same exit-status meaning, and only tools present on a minimal install of all three targets. Extend bin/build's report self-test to the new fields.

### Documentation

- en/docs/contracts.md: contract 0.3.0, the new path in uc-contracts, the shared parameter `initial_password` and where the hash comes from, the coverage table updated (RN-ACC-03 and RN-FWM-01 shown as retired, rows for RN-ACC-06, RN-ACC-07, RN-ACC-08 and RN-SSH-03), and the per-node checks updated. Remove the observation about MAC addresses and machine IDs from "Observations for uc-contracts" once the report quotes them, and say so.
- en/docs/raw-node-baseline.md and the target docs: anything that describes password login over SSH.
- README.md: the link to the contract.

## Working rules

- Do not commit. Leave all changes for my review.
- Ask before adding dependencies or changing anything outside this repository.
- Record this session in en/docs/sessions/ as `<timestamp>-session.md`, following en/docs/sessions/README.md.

## Done when

- bin/build passes for all three targets, with self-tests OK and shellcheck clean.
- sudo bin/build --check-firewall-only passes.
- You give me a table with one row per contract 0.3.0 clause and confirmation field and one column per target. Each cell says met (and how), not met (and why), or "needs an install to confirm". After it, list the checks I should run on a real installed node of each target, including from another machine: key login works, password login is refused for `initial` and for `root`, and `sudo -k; sudo true` asks for the password.
- I can rebuild the Debian 13 metal node (deb13-vt) with a batch hash from uc-controller, so that the controller's admission kickoff has a contract 0.3.0 node to test against.
