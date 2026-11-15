# Building a Verified Ventoy Install Stick for Ubuntu Deployment

## Notes on Usage

While this is a lengthy document, once you understand how Ventoy works it is very easy to use
for custom-installing many operating systems — including macOS, Linux, and Windows — by
leveraging **VentoyPlugson** to generate a `ventoy.json` file (which can also be edited by
hand). See [§6.3 VentoyPlugson](#63-generate-ventoyjson-with-ventoyplugson-gui).

## Description

A technical walkthrough: downloading and **verifying** Ventoy from source, installing it to
USB on Linux, laying out an ISO library, cryptographically verifying the OS images, and
wiring up an unattended Ubuntu autoinstall from a `nocloud` seed.

This is the engineer's version of the process — every artifact is verified before it runs,
and the Secure Boot trade-offs are stated rather than glossed over.

---

## Scope and trust model

Ventoy is convenient but only *partially* reproducible: its source tree ships prebuilt binary
blobs, and its Secure Boot support works by enrolling a Machine Owner Key (MOK) and then
chaining unsigned GRUB2 — which weakens the chain it appears to support. None of that is fatal
for an **install-time** tool you boot transiently, but it means two habits are non-negotiable:

1. **Get Ventoy only from the official source** (`github.com/ventoy/Ventoy` or `ventoy.net`)
   and check its published hash.
2. **Verify every OS image** against the distro's signature before it ever boots — and don't
   assume you already hold a trusted copy of the signing key (see [§5](#5-verify-the-os-images)).

For Secure-Boot-critical targets (a hardened host, anything using TPM-backed full-disk
encryption), consider skipping Ventoy's MOK enrollment entirely — see
[§3, Secure Boot](#secure-boot-decision).

Versions referenced: **Ventoy 1.1.12** (released 2026-04-23) and **Ubuntu 24.04 / 26.04**.

---

## Prerequisites

- A Linux machine to prepare the stick (`wget`, `curl`, `tar`, `gpg`, `sha256sum`, `lsblk` — all in base installs).
- A USB stick large enough for your ISO library (a fast, high-capacity stick is worth it).
- The target ISOs (Ubuntu Server/Desktop, etc.).

> **Destructive operation ahead.** Installing Ventoy *erases the target device*. Identify the
> correct device node carefully (§3) — getting it wrong overwrites the wrong disk.

---

## 1. Download Ventoy from source

Pull Ventoy only from the official GitHub releases or `ventoy.net`. Never use third-party
"download portals" — that is exactly the wrong place for a boot-level tool.

### 1.1 Latest stable release (recommended)

GitHub's `releases/latest` endpoint returns the newest release that is not a draft or
pre-release — i.e. the latest *stable* one, since Ventoy flags betas as pre-releases.

#### Resolve the latest tag (no API, not rate-limited)

```bash
tag=$(curl -fsSLI -o /dev/null -w '%{url_effective}' \
  https://github.com/ventoy/Ventoy/releases/latest | sed 's#.*/tag/##')   # -> v1.1.12
ver=${tag#v}
wget "https://github.com/ventoy/Ventoy/releases/download/${tag}/ventoy-${ver}-linux.tar.gz"
```

This leans on Ventoy's stable asset naming (`ventoy-<ver>-linux.tar.gz`) and avoids the API
rate limit entirely.

#### Via the GitHub API (asset-name robust)

The API approach finds the right asset by name instead of constructing the URL, so it survives
a maintainer renaming things:

```bash
url=$(curl -fsSL https://api.github.com/repos/ventoy/Ventoy/releases/latest \
  | jq -r '.assets[] | select(.name | endswith("linux.tar.gz")) | .browser_download_url')
wget "$url"
```

One gotcha: the unauthenticated GitHub API is rate-limited to 60 requests/hour per IP. If you
will call it repeatedly (CI, a refresh script), either authenticate with a token
(`-H "Authorization: Bearer $GITHUB_TOKEN"`) or use the tag-resolution method above.

### 1.2 A specific pinned release

```bash
cd ~/Downloads
VTOY=1.1.12
wget "https://github.com/ventoy/Ventoy/releases/download/v${VTOY}/ventoy-${VTOY}-linux.tar.gz"
```

If your network blocks GitHub release CDNs, the same artifact is mirrored from the download
page linked off `https://www.ventoy.net`.

> Whichever method you use, always verify the hash (§2) **before** extracting — "latest" tells
> you the file is current, not that it is untampered.

---

## 2. Verify the Ventoy download

Ventoy doesn't publish a widely-distributed detached GPG signature the way Ubuntu does, so the
practical integrity check is the **SHA-256 published on the official release page**. The
checksums live on the release page (for v1.1.12, `releases/tag/v1.1.12`, including the
`sha256.txt` asset).

Published checksums for v1.1.12 (confirm these against the official release page before
trusting them):

```text
04620b546bcc5eeeb5971767595b3713ee3de71580a82449053c53a7cb32fcd9  ventoy-1.1.12-linux.tar.gz
c1345277c8f3a8fcc213945687bcad5ff1d680b1d6b7b7a1cdf85f7d4a368406  ventoy-1.1.12-livecd.iso
e70c505be08d99c55e506832f596c430a9c36f8d087f25542d3f6d332d9b6473  ventoy-1.1.12-windows.zip
```

Compute the hash locally and compare it byte-for-byte:

```bash
sha256sum ventoy-1.1.12-linux.tar.gz
```

Expected output:

```text
04620b546bcc5eeeb5971767595b3713ee3de71580a82449053c53a7cb32fcd9  ventoy-1.1.12-linux.tar.gz
```

Only proceed if they match. Then unpack:

```bash
tar -xf ventoy-1.1.12-linux.tar.gz
cd ventoy-1.1.12
```

---

## 3. Install Ventoy to the USB

First, **identify the device unambiguously**. Unplug other USB disks, then:

```bash
lsblk -o NAME,SIZE,MODEL,TRAN,MOUNTPOINT
# The USB stick is typically the device with TRAN=usb, e.g. /dev/sdb (NOT a partition like sdb1).
```

Install Ventoy onto the whole device (replace `/dev/sdX` with your actual node):

```bash
sudo sh Ventoy2Disk.sh -i /dev/sdX
```

Useful flags:

| Flag | Purpose |
|------|---------|
| `-i` | Install Ventoy (fails if Ventoy is already present — use `-I` to force). |
| `-I` | Force (re)install, **wiping the device**. |
| `-u` | Update Ventoy in place, **keeping** your ISOs and data. |
| `-s` | Enable Secure Boot support (enrolls a MOK on first boot — see below). |
| `-g` | Use a GPT partition layout (default is MBR; GPT is fine on modern UEFI). |
| `-L` | Set the data-partition label. |

After install you'll have two partitions: a large exFAT data partition (where your ISOs go)
and a small EFI partition (Ventoy's bootloader). To update Ventoy later without losing your
library, use `sudo sh Ventoy2Disk.sh -u /dev/sdX`.

![ventoy-usb-partitions](./building-a-verified-ventoy-install-stick-for-ubuntu-deployment.assets/ventoy-usb-partitions.png)

So in this case:

```bash
sudo sh Ventoy2Disk.sh -u /dev/sdb
```

Output example:

```text
 sudo sh ./Ventoy2Disk.sh -u /dev/sdb
**********************************************
      Ventoy: 1.1.12  x86_64
      longpanda admin@ventoy.net
      https://www.ventoy.net
**********************************************
Upgrade operation is safe, all the data in the 1st partition (iso files and other) will be unchanged!
Update Ventoy  1.0.96 ===> 1.1.12   Continue? (y/n)
esp partition processing ...
Update Ventoy on /dev/sdb successfully finished.
```

### Secure Boot decision

`-s` makes Ventoy bootable under Secure Boot by enrolling **Ventoy's own key** via MOKManager
the first time you boot it on a machine. That key then persists in that machine's firmware
until removed (`mokutil --delete`), and the chain after the signed shim is unsigned GRUB2 — so
this trades away part of Secure Boot's guarantee.

For most install jobs the cleaner path is:

- **Temporarily disable Secure Boot** in firmware → install Ventoy *without* `-s` → boot the
  installer → install the OS → **re-enable Secure Boot**. The *installed* Ubuntu uses
  Canonical's properly signed shim chain, so the running system keeps full Secure Boot
  integrity and you never touch the machine's MOK list.
- Reserve `-s` for machines where you can't toggle Secure Boot in firmware.

---

## 4. Lay out the ISO library

Mount the Ventoy data partition and copy ISOs in. Ventoy scans subdirectories, so organize
freely. A working layout:

```text
/media/<you>/Ventoy/
├── iso
│   ├── bios                     # vendor BIOS/firmware update ISOs
│   ├── qubes
│   │   └── Qubes-R4.2.4-x86_64.iso
│   ├── ssh
│   │   └── dropbear.pub         # pubkey staged for injection into installs
│   ├── ubuntu
│   │   ├── 24.04
│   │   │   ├── SHA256SUMS       # checksums + signature kept beside the ISOs
│   │   │   ├── SHA256SUMS.gpg
│   │   │   ├── ubuntu-24.04.2-desktop-amd64.iso
│   │   │   ├── ubuntu-24.04.2-live-server-amd64.iso
│   │   │   └── ubuntu-24.04-autoinstall.iso # self-contained custom ISO (alt. method)
│   │   ├── 26.04
│   │   │   ├── SHA256SUMS
│   │   │   ├── SHA256SUMS.gpg
│   │   │   └── ubuntu-26.04-live-server-amd64.iso
│   │   └── mini.iso
│   ├── ventoy
│   │   └── ventoy-1.0.96-livecd.iso
│   └── windows
│       └── Win11_24H2_English_x64.iso
├── nocloud                          # autoinstall seeds, mirroring the iso/ tree (see §6)
│   └── ubuntu
│       └── 26.04
│           ├── server
│           │   ├── meta-data
│           │   └── user-data
│           └── desktop
│               ├── meta-data
│               └── user-data
└── ventoy
    └── ventoy.json                  # Ventoy plugin config
```

Keeping `SHA256SUMS` and `SHA256SUMS.gpg` *next to* each release is good practice — it makes
the verification in §5 repeatable any time, directly from the stick. Grab the matching pair for
each release from `https://releases.ubuntu.com/<release>/`. Note, though, that these on-stick
copies are only a convenience cache: your trust still derives from the verified signing key and
signature (§5), not from the fact that the files sit on the stick.

---

## 5. Verify the OS images

A multiboot stick is a tampering target, so verify each Ubuntu image against Canonical's
signature *before* trusting it. The chain has three links:

1. You trust Canonical's image-signing **public key**.
2. That key verifies the **signature** on `SHA256SUMS`.
3. `SHA256SUMS` verifies the **ISO**.

Most guides quietly assume link 1 is already solved. It usually isn't — and that is the part
worth slowing down for.

### 5.1 The trust-bootstrap problem (chicken-and-egg)

To verify `SHA256SUMS.gpg`, you need Canonical's signing key. But how do you trust *that key*?
Pulling it from a keyserver only proves "a key with this ID exists" — not that it is
Canonical's. An attacker who can tamper with your download can equally hand you a key that
"verifies" their forged ISO perfectly.

Canonical's own guidance frames this directly: establishing trust in the signing key is a
chain-of-trust problem, and the convenient "it's already locally signed, so it's trusted"
route is **circular** — it only works if you established that trust on a previous occasion.
The first time, you must bootstrap trust by some independent means.

In practice, bootstrap trust by **confirming the key's fingerprint out-of-band**: compare the
fingerprint you obtain against Canonical's published value, fetched over HTTPS from at least
one independent, authoritative source (the Ubuntu security documentation and the Ubuntu
verification tutorial), and ideally cross-checked against the keyring shipped with an already-trusted
Ubuntu install. The target fingerprint for the *Ubuntu CD Image Automatic Signing Key (2012)* is:

```text
8439 38DF 228D 22F7 B374 2BC0 D94A A3F0 EFE2 1092
```

Only once you have confirmed that fingerprint should you use the key to verify anything.

### 5.2 Get and authenticate Canonical's signing key

**Obtain the keyring.** Download Canonical's keyring (which includes the CD Image signing key)
into a *local, isolated* keyring file rather than polluting your default keyring:

```bash
curl -fsSL -o ubuntu.gpg \
  "https://archive.ubuntu.com/ubuntu/project/ubuntu-archive-keyring.gpg"
```

**Authenticate it.** List the keys in that file and confirm the CD Image key's fingerprint
matches the value from §5.1:

```bash
gpg --no-default-keyring --keyring ./ubuntu.gpg \
    --keyid-format long --list-keys --with-fingerprint
# Confirm the "Ubuntu CD Image Automatic Signing Key (2012)" fingerprint is:
#   8439 38DF 228D 22F7 B374 2BC0 D94A A3F0 EFE2 1092
# and cross-check that value against Canonical's HTTPS-served docs (see References).
```

*Keyserver alternative.* You can instead fetch the key by fingerprint from a keyserver — but
this does **not** bootstrap trust on its own; you must still verify the fingerprint out-of-band
exactly as above:

```bash
gpg --keyid-format long --keyserver hkp://keyserver.ubuntu.com \
    --recv-keys 0x843938DF228D22F7B3742BC0D94AA3F0EFE21092
gpg --fingerprint 0x843938DF228D22F7B3742BC0D94AA3F0EFE21092   # then compare, as above
```

### 5.3 Get the checksums for your release

```bash
export RELEASE=resolute          # 26.04 (use 'noble' for 24.04)
curl -fsSL -O "https://releases.ubuntu.com/$RELEASE/SHA256SUMS"
curl -fsSL -O "https://releases.ubuntu.com/$RELEASE/SHA256SUMS.gpg"
```

Because trust derives from the signature rather than the transport, these files don't strictly
need a secure channel — but use HTTPS anyway. (If you already staged `SHA256SUMS`/`.gpg` beside
the ISOs on the stick, you can verify those copies in place instead.)

### 5.4 Verify the signature, then the ISO

```bash
cd /media/<you>/Ventoy/iso/ubuntu/24.04   # or wherever the ISO + SHA256SUMS live

# (a) Is SHA256SUMS genuinely signed by the key you authenticated in 5.2?
gpg --no-default-keyring --keyring ~/ubuntu.gpg \
    --keyid-format long --verify SHA256SUMS.gpg SHA256SUMS
#   -> expect: Good signature from "Ubuntu CD Image Automatic Signing Key (2012)"

# (b) Does the ISO match the signed checksum?
sha256sum -c SHA256SUMS --ignore-missing
#   -> expect: ubuntu-24.04.2-live-server-amd64.iso: OK
```

A `Good signature` plus `OK` means the image is authentic. Because the signature is what's
trusted, even an ISO and checksum fetched over plain HTTP cannot be forged without Canonical's
private key. Anything else — `BAD signature`, no `OK`, or a fingerprint that didn't match in
§5.2 — means stop: re-download, and do not boot.

> If you verified against the local `ubuntu.gpg` keyring you will *not* see a "key not
> certified" warning. If you imported the key into your default keyring instead, that warning
> is normal and simply means you haven't locally signed the key. You can silence it for future
> runs by importing and locally signing the key (`gpg --import ./ubuntu.gpg` then
> `gpg --lsign-key 0x843938DF228D22F7B3742BC0D94AA3F0EFE21092`) — which is also what makes the
> "prior signature" shortcut from §5.1 available next time.

---

## 6. Wire up the unattended Ubuntu autoinstall

Many distros provide an unattended-deployment mechanism driven by a script or template —
`unattend.xml` for Windows, kickstart for RHEL/CentOS, AutoYaST for SUSE, preseed for Debian,
and cloud-init **autoinstall** for modern Ubuntu. You have two complementary paths on the stick.

### 6.1 Path A — pre-baked custom ISO

`ubuntu-24.04-autoinstall.iso` is a self-contained custom ISO with the answer file baked in. It
boots and installs with zero extra config, which makes it the best choice when you want a
frozen, reproducible artifact — and for Secure-Boot-critical installs, since no Ventoy plugin
layer is involved.

> **Licence note.** Building a remastered ISO for your *own* use is fine. Publicly
> redistributing a modified Canonical image can conflict with Ubuntu's trademark policy (the
> Ubuntu name and logos are trademarked), so keep custom ISOs internal or strip the trademarks
> before sharing.

### 6.2 Path B — stock ISO + `nocloud` seed (DRY)

Keep the official `26.04` server ISO untouched and point Ventoy at your editable `user-data`.
Edit the YAML, reboot, reinstall — no remastering required, which also sidesteps the
redistribution concern above and keeps your iteration loop fast. Ventoy injects the autoinstall
via its plugin config, generated most reliably with VentoyPlugson (next).

**Seed layout.** Keep the seeds under `nocloud/` (never loose in the ISO directories — that
dir is for verification artifacts like `SHA256SUMS`), and give each release/variant its own
leaf directory that *mirrors* the `iso/` tree:

```text
nocloud/ubuntu/26.04/server/{user-data,meta-data}
nocloud/ubuntu/26.04/desktop/{user-data,meta-data}
```

Two reasons for the leaf-directory shape rather than one shared file. First, it serves **both**
delivery methods: Ventoy's `template` accepts any file path, but the boot-arg fallback
(`ds=nocloud;s=<dir>`) needs a *directory* containing files named exactly `user-data` and
`meta-data` — so a per-variant directory satisfies both at once. Second, server and desktop
genuinely need different answers (storage, the desktop-autoinstall quirks), so they belong in
separate seeds. Differentiate by **directory, not filename** — the files must stay named
`user-data`/`meta-data`. Each leaf carries its own trivial `meta-data` (just an `instance-id`),
and each `user-data` is generated from the one hardened base rather than hand-copied.

### 6.3 Generate `ventoy.json` with VentoyPlugson (GUI)

Hand-writing the `auto_install` plugin config is error-prone, and its schema has shifted across
Ventoy versions. **VentoyPlugson** removes the guesswork: it's a small web GUI (bundled in the
same `ventoy-1.1.12-linux.tar.gz`) that writes a schema-correct `ventoy.json` for the exact
Ventoy version on your stick. Available since Ventoy 1.0.62, it *only* generates `ventoy.json`,
and on every launch it backs up the current one as `ventoy_backup.json`, so edits are reversible.

#### Launch it (Linux)

With the Ventoy stick plugged in, from the extracted `ventoy-1.1.12/` directory:

```bash
# Identify the Ventoy device (the whole disk, e.g. /dev/sdb — not a partition)
lsblk -o NAME,SIZE,MODEL,TRAN,LABEL

# Start the configurator against that device
sudo bash VentoyPlugson.sh /dev/sdX
```

Then open a browser to:

```text
http://127.0.0.1:24681
```

The server binds to localhost only. Leave the terminal running while you edit; stop it with
`Ctrl-C` when finished.

#### Configure the Auto Installation plugin

In the web UI, open the **Auto Install Plugin** and click **Add**. When prompted, choose
**`[image]`** (set the template for one file), *not* `[parent]` — `[parent]` blankets every ISO
under a directory with the same answer file, which on a multiboot stick would wrongly target
`mini.iso`, the already-self-contained `*-autoinstall.iso`, etc. Add one `[image]` entry per ISO
you want automated, each pointing its template at the matching per-variant seed:

| Image | Template |
|-------|----------|
| `/iso/ubuntu/26.04/ubuntu-26.04-live-server-amd64.iso` | `/nocloud/ubuntu/26.04/server/user-data` |
| `/iso/ubuntu/24.04/ubuntu-24.04.2-desktop-amd64.iso` (optional) | `/nocloud/ubuntu/26.04/desktop/user-data` |

All paths are relative to the Ventoy partition root with a leading slash (`/iso/...`,
`/nocloud/...`) — not the `/media/...` mount path. A green **OK** beside both the image and the
template means Plugson resolved both paths on the stick (the template OK confirms the
`user-data` file actually exists where you put it). Each change saves to `ventoy.json`
immediately — there is no separate "save" step.

Leave the **timeout** and **autosel** checkboxes *off* for now. `autosel` makes Ventoy
auto-pick and boot this ISO unattended — convenient, but on a stick that also boots Qubes and
Windows, an auto-selected destructive install is a foot-gun. Turn it on only after a clean
test-boot.

#### Resulting `ventoy.json`

The GUI produces something equivalent to this (verify it matches after editing):

```json
{
  "auto_install": [
    {
      "image": "/iso/ubuntu/26.04/ubuntu-26.04-live-server-amd64.iso",
      "template": "/nocloud/ubuntu/26.04/server/user-data"
    },
    {
      "image": "/iso/ubuntu/24.04/ubuntu-24.04.2-desktop-amd64.iso",
      "template": "/nocloud/ubuntu/26.04/desktop/user-data"
    }
  ]
}
```

#### Manual edits and fallback

Because Plugson emits the schema for the Ventoy version on your stick, it is the authoritative
reference — but if you edit `ventoy.json` by hand, confirm the keys against the current plugin
reference at `https://www.ventoy.net/en/plugin_autoinstall.html`.

If the plugin entry doesn't take, fall back to the boot-time kernel argument: at the Ventoy
menu, edit the boot entry and add `autoinstall ds=nocloud;s=/nocloud/ubuntu/26.04/server/`
(point `s=` at the *directory* holding `user-data` + `meta-data`), which is the mechanism
subiquity ultimately consumes. Without the `autoinstall` argument, the installer simply waits
for a human.

The seed installs a hardened base: a single admin user `initial`, key-only SSH, `ufw` locked to
(rate-limited) port 22, sshd hardening, and fail2ban — with Incus deliberately *not* baked in
(it's a separate first-boot overlay). The install also logs `host` / `ip` / `user` / public key
to `install-ip.log` on the Ventoy stick via `late-commands`, so a headless box reports its
address back without a monitor or network scan. The `meta-data` file can be empty but must
exist.

#### Notes for the desktop ISO

The same `nocloud/user-data` drives both, but desktop autoinstall is newer and more limited
than Server's. The Ubuntu **desktop** installer expects to be told to run unattended, so if the
desktop entry boots into the interactive installer instead, use the boot-time `autoinstall`
fallback above and confirm the desktop ISO's autoinstall support for that release. Server is the
more reliable autoinstall target of the two.

#### After configuring

Stop the Plugson server (`Ctrl-C`), then verify outside the GUI that it wrote what you expect:

```bash
cat /media/<you>/Ventoy/ventoy/ventoy.json          # auto_install: image -> template
ls -l /media/<you>/Ventoy/ventoy/ventoy_backup.json  # proof the prior config was backed up
sync                                                 # flush exFAT before pulling the stick
```

> Plugson's green **OK** only confirms the *paths* resolve — it does **not** validate the YAML
> inside `user-data`, so a syntax slip or a still-unreplaced `$6$REPLACE_WITH_GENERATED_HASH`
> won't surface here; it surfaces mid-install. Do the VM dry-run in §7 before bare metal.

---

## 7. Boot, install, and validate

1. Boot the target from the Ventoy stick (or chain it from Windows — Appendix A).
2. Select the ISO. With the autoinstall wired up, the install proceeds unattended.
3. Find the box (e.g. `install-ip.log` on the stick, or `raw.local`) and SSH in as `initial`
   with your key; confirm `ufw status`.

**Test before you flash a real machine.** Validate the whole autoinstall in a VM first so you
only touch bare metal once it's clean:

```bash
incus launch images:ubuntu/26.04/cloud testbox --vm \
  -c cloud-init.user-data="$(cat nocloud/ubuntu/26.04/server/user-data)"
incus exec testbox -- cloud-init status --wait
```

---

## Appendix A — No USB boot? Chain Ventoy from the Windows boot manager

On a UEFI machine that won't boot from USB but already runs Windows, you can register Ventoy's
EFI loader in the Windows boot manager and chain into it.

1. Boot Windows. Mount the EFI System Partition and copy Ventoy's EFI files from the stick:

   ```bat
   mountvol S: /S
   xcopy X:\EFI\Boot S:\EFI\Ventoy /E /H /K
   ```

   (`X:` = Ventoy EFI partition, `S:` = ESP.)

2. Add a boot entry pointing at Ventoy's loader (administrator command prompt):

   ```bat
   bcdedit /copy {bootmgr} /d "Ventoy"
   bcdedit /set {guid} path \EFI\Ventoy\ventoyx64.efi
   ```

   (Substitute the `{guid}` returned by the `/copy` command.)

3. Reboot, choose **Ventoy** in the Windows boot menu, then pick your ISO.

This avoids needing PXE or VHD boot. Note it inherits the same Secure Boot considerations from
§3.

---

## Appendix B — Security checklist

- [ ] Ventoy pulled only from `github.com/ventoy` / `ventoy.net`.
- [ ] Ventoy tarball SHA-256 matches the official release page.
- [ ] Secure Boot toggled in firmware for the install (no MOK enrollment) where feasible.
- [ ] Canonical signing-key fingerprint confirmed out-of-band before first use (§5.1–5.2).
- [ ] Every OS ISO verified: `gpg --verify` → `Good signature`, `sha256sum -c` → `OK`.
- [ ] `SHA256SUMS` + `.gpg` kept beside each release on the stick.
- [ ] Autoinstall validated in a VM before bare-metal use.
- [ ] Network/PXE (iVentoy) avoided except on a fully controlled, isolated LAN.

---

## References

- Ventoy — official site and downloads: <https://www.ventoy.net>
- Ventoy source and releases: <https://github.com/ventoy/Ventoy>
- VentoyPlugson (GUI config generator): <https://www.ventoy.net/en/plugin_plugson.html>
- Ventoy auto-install plugin: <https://www.ventoy.net/en/plugin_autoinstall.html>
- How to verify your Ubuntu download (Canonical): <https://ubuntu.com/tutorials/how-to-verify-ubuntu>
- Ubuntu image integrity verification, incl. the trust chain-of-trust / chicken-and-egg discussion (Canonical security docs): <https://documentation.ubuntu.com/security/software-integrity/image-verification/>
- Ubuntu autoinstall reference: <https://canonical-subiquity.readthedocs-hosted.com/en/latest/reference/autoinstall-reference.html>

---

*Draft walkthrough — adapt the device nodes, paths, and `ventoy.json` schema to your setup
before relying on it. Released under CC BY-SA 4.0; fill in your own attribution line.*
