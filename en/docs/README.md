# uc-ventoy documentation

uc-ventoy generates Ventoy answer files that install any supported Linux distribution as a raw node with an identical, Ansible-ready SSH interface.

## Read in this order

1. [contracts.md](contracts.md): the raw node contract uc-ventoy implements as a producer, which targets implement which version, how each clause is met, and the checks to run on an installed node.
2. [raw-node-baseline.md](raw-node-baseline.md): what a raw node is (requirements RN-01 to RN-12), the inputs, the canonical files, acceptance checks, the handover to Ansible, and how Ventoy delivers it all.
3. The target document for the distribution you install:
   - [debian-13.md](debian-13.md): Debian 13, Debian Installer preseed.
   - [ubuntu-24.04.md](ubuntu-24.04.md): Ubuntu Server 24.04 LTS, Subiquity autoinstall, also usable as a Linux Mint 22 base.
   - [ubuntu-26.04.md](ubuntu-26.04.md): Ubuntu Server 26.04 LTS, Subiquity autoinstall.
4. [iso-downloads.md](iso-downloads.md): where the ISOs come from and how to verify them before copying them to the stick.

## Also here

- [notes/](notes/): research notes that inform future targets, such as [btrfs-support-by-distro.md](notes/btrfs-support-by-distro.md).
- [sessions/](sessions/): records of the working sessions that produced the generator and these documents.

## Conventions

- Each document's version and status are in its front matter. Filenames carry no version; git keeps the history.
- Documents link to files in the repository rather than copying them, so they cannot drift from what is installed.
