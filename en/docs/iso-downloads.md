---
title: "ISO Downloads"
version: "0.3.0"
date: 2026-11-14
author: "Christopher Steel"
status: "Draft"
description: "Where uc-ventoy's installation ISOs come from, how their checksums and signatures are verified, and how they reach the Ventoy stick."
---

# ISO Downloads

ISOs are downloaded and verified in a local store, `~/iso`, laid out by distribution and version, and only then copied to the Ventoy stick. `bin/check-isos` verifies the store. Nothing is hashed on the stick itself.

```text
~/iso/
├── debian/13/
└── ubuntu/
    ├── 24.04/
    └── 26.04/
```

## Signing keys

Import each key once. gpg then verifies future downloads from the same project without fetching the key again.

| Project | Key | Fingerprint | Keyserver |
| --- | --- | --- | --- |
| Debian | Debian CD signing key | `DF9B 9C49 EAA9 2984 3258 9D76 DA87 E80D 6294 BE9B` | `keyring.debian.org` |
| Ubuntu | Ubuntu CD Image Automatic Signing Key (2012) | `8439 38DF 228D 22F7 B374 2BC0 D94A A3F0 EFE2 1092` | `keyserver.ubuntu.com` |

Debian publishes its fingerprint on its verification page <a id="cite-debian-verify"></a>([Debian Project, n.d.](#ref-debian-verify)), and Canonical publishes Ubuntu's in its verification tutorial <a id="cite-canonical-verify"></a>([Canonical, n.d.](#ref-canonical-verify)). Pass the fingerprint to gpg as one word, without spaces. After a successful `gpg --verify`, the warning that the key "is not certified with a trusted signature" is normal: it only means you have not certified the key yourself. What matters is that the primary key fingerprint gpg prints matches the table.

## Debian 13

Mirror: University of Waterloo Computer Science Club. Debian publishes `SHA512SUMS` and its signature `SHA512SUMS.sign` beside the ISOs.

```bash
MIRROR="https://mirror.csclub.uwaterloo.ca/debian-cd/current/amd64/iso-cd"
cd ~/iso/debian/13

# Grab checksums + signature
wget -N "${MIRROR}/SHA512SUMS" \
        "${MIRROR}/SHA512SUMS.sign"

# Grab the netinst ISO named in SHA512SUMS
ISO="$(grep -o 'debian-13\.[0-9.]*-amd64-netinst\.iso' SHA512SUMS | sort -V | tail -n 1)"
echo "$ISO"
wget -c "${MIRROR}/${ISO}"

# Verify GPG signature (proves the checksum file is authentic)
gpg --keyserver keyring.debian.org --recv-keys DF9B9C49EAA9298432589D76DA87E80D6294BE9B
gpg --verify SHA512SUMS.sign SHA512SUMS

# Verify checksum (only the ISO downloaded here)
sha512sum -c --ignore-missing SHA512SUMS
```

`current` follows the newest Debian release. When Debian 14 is released, it stops serving Debian 13 ISOs, and the `grep` finds nothing.

## Ubuntu 24.04 and 26.04

Primary mirror: Xenyth Cloud Public Mirror, Toronto. Fallback: University of Waterloo Computer Science Club. Ubuntu publishes `SHA256SUMS` and its signature `SHA256SUMS.gpg`.

The version folders, `24.04` and `26.04`, always point to the newest point release. A mirror's `SHA256SUMS` can list several point releases and the desktop images as well, so the script picks the newest server ISO. The 24.04 folder is currently labelled 24.04.5.1 because of a desktop-only respin: Canonical reissued the desktop image after an installer crash and left the server image unchanged <a id="cite-linuxcompatible-2026"></a>([Linux Compatible, 2026](#ref-linuxcompatible-2026)). The server ISO is still `ubuntu-24.04.5-live-server-amd64.iso`.

```bash
VERSION="24.04"     # or 26.04
MIRROR="https://mirror.xenyth.net/ubuntu-releases/${VERSION}"
# MIRROR="https://mirror.csclub.uwaterloo.ca/ubuntu-releases/${VERSION}"
cd ~/iso/ubuntu/${VERSION}

# Grab checksums + signature
wget -N "${MIRROR}/SHA256SUMS" \
        "${MIRROR}/SHA256SUMS.gpg"

# Grab the newest server ISO named in SHA256SUMS
ISO="$(grep -o "ubuntu-${VERSION}[0-9.]*-live-server-amd64\.iso" SHA256SUMS | sort -V | tail -n 1)"
echo "$ISO"
wget -c "${MIRROR}/${ISO}"

# Verify GPG signature (proves the checksum file is authentic)
gpg --keyserver keyserver.ubuntu.com --recv-keys 843938DF228D22F7B3742BC0D94AA3F0EFE21092
gpg --verify SHA256SUMS.gpg SHA256SUMS

# Verify checksum (only the ISO downloaded here)
sha256sum -c --ignore-missing SHA256SUMS
```

The dot in `${VERSION}` matches any character in the `grep` pattern, which is harmless for these filenames.

## Checking the store

`bin/check-isos` rehashes every ISO on every run and accepts any directory, so a new download can be checked on its own:

```sh
bin/check-isos                        # everything under ~/iso
bin/check-isos ~/iso/ubuntu/24.04     # one folder
```

It uses `SHA512SUMS` when a folder has one, otherwise `SHA256SUMS`, checks the matching `.sign` or `.gpg` signature when gpg and the key are available, and reports each ISO as `OK`, `FAILED` or `UNVERIFIED`. It exits with an error if any ISO fails, so it can gate the copy to the stick. `--strict` also fails on unverified ISOs and unchecked signatures.

## Copying to the stick

Copy each verified ISO into the matching folder on the stick, for example `/media/initial/Ventoy/iso/ubuntu/24.04/`, and delete the ISO it replaces so that each version folder holds one ISO. Ventoy matches the folder, so no rebuild is needed when only the ISO changes.

## References

<a id="ref-canonical-verify"></a>
Canonical. (n.d.). *How to verify your Ubuntu download*. Ubuntu tutorials. Retrieved November 14, 2026, from https://ubuntu.com/tutorials/how-to-verify-ubuntu
[Return to citation](#cite-canonical-verify)

<a id="ref-debian-verify"></a>
Debian Project. (n.d.). *Verifying authenticity of Debian images*. Retrieved November 14, 2026, from https://www.debian.org/CD/verify
[Return to citation](#cite-debian-verify)

<a id="ref-linuxcompatible-2026"></a>
Linux Compatible. (2026). *Ubuntu Linux 24.04.5.1 LTS fixes desktop installer crash in emergency rebuild*. https://www.linuxcompatible.org/story/ubuntu-linux-240451-lts-fixes-desktop-installer-crash-in-emergency-rebuild/
[Return to citation](#cite-linuxcompatible-2026)
