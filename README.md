# ventoy-raw-node

Generator for Ventoy answer files that build a raw node: the minimal, identical starting state of every server before Ansible takes over. See [en/docs/raw-node-baseline.md](en/docs/raw-node-baseline.md) for the definition, and [en/docs/contracts.md](en/docs/contracts.md) for how every node fulfils the raw node contract, version 0.3.0. The contract itself lives in uc-contracts: [`uc-raw-node-contract.md`](https://github.com/steelcj/uc-contracts/blob/main/en/docs/node-onboarding-pipeline/1-raw-node-egress/uc-raw-node-contract.md).

Run it on your workstation. Only its output goes on the Ventoy stick.

## Usage

Use this to quickly generate new automated configurations for multiple linux versions quickly and efficiently.

## Recommendations

I would highly recommend running the included utility against any ISO's that you will add your Ventoy or manually verifying their checksums before using them for installations.

### bin/check-isos

bin/check-isos is used on your local system to check a directory of one or more iso images you have downloaded to  your local system. 

**`bin/check-isos`** takes any directory and rehashes every ISO on every run:

```sh
bin/check-isos                        # everything under ~/iso
```

Output example:

```bash
/home/initial/iso/bios?
  UNVERIFIED  n32ur32w.iso  (no SHA512SUMS or SHA256SUMS in this folder)
  UNVERIFIED  n32ur33w.iso  (no SHA512SUMS or SHA256SUMS in this folder)

/home/initial/iso/debian/13
  SIGNED      SHA512SUMS: SHA512SUMS.sign good signature "Debian CD signing key <debian-cd@lists.debian.org>" [unknown]
  OK          debian-13.7.0-amd64-netinst.iso

/home/initial/iso/ubuntu/24.04
  SIGNED      SHA256SUMS: SHA256SUMS.gpg good signature "Ubuntu CD Image Automatic Signing Key (2012) <cdimage@ubuntu.com>" [unknown]
  OK          ubuntu-24.04.5-live-server-amd64.iso

/home/initial/iso/ubuntu/26.04
  SIGNED      SHA256SUMS: SHA256SUMS.gpg good signature "Ubuntu CD Image Automatic Signing Key (2012) <cdimage@ubuntu.com>" [unknown]
  FAILED      ubuntu-26.04.1-live-server-amd64.iso  (SHA256 does not match)

/home/initial/iso/windows
  UNVERIFIED  Win11_24H2_English_x64.iso  (no SHA512SUMS or SHA256SUMS in this folder)
  UNVERIFIED  Win11_25H2_English_x64_v2.incus.iso  (no SHA512SUMS or SHA256SUMS in this folder)
  UNVERIFIED  Win11_25H2_English_x64_v2.iso  (no SHA512SUMS or SHA256SUMS in this folder)
  UNVERIFIED  virtio-win-0.1.302.iso  (no SHA512SUMS or SHA256SUMS in this folder)

Summary: 2 OK, 1 failed, 6 unverified, 0 checksum files without a checked signature
```

The script works by looking for checksum files in each iso directories in order of preference:

```bash
CHECKSUM_FILES = (("SHA512SUMS", "sha512"), ("SHA256SUMS", "sha256"))
SIGNATURE_SUFFIXES = (".gpg", ".sign")
```

Directories containing the iso and checksums are verified and every iso contained in the root directory scanned.  **`check-isos`** reports on each ISO in the scanned directory tree  `OK`, `FAILED` or `UNVERIFIED`, and the tool exits with an error if any ISO fails.

So in the example output above we see that in ~/iso the ubuntu/24/ubuntu-26.04.1-live-server-amd64.iso failed the checksum and the process stopped. In addition we see that 6 iso images where unverified meaning to no checksums where included in each of those iso directories. The scanner also:

- Catches a corrupted ISO and a checksum file edited after signing
- Lists all ISOs with no checksum to check against

In this example you might download a new ubuntu/26/ server iso and the associated checksum files to the directory. In this case you might only want to scan that director so you would run:

```bash
bin/check-isos ~/iso/ubuntu/24.04     # just a newly downloaded iso
```

I did this using a script which output this:

```bash
gpg: key D94AA3F0EFE21092: "Ubuntu CD Image Automatic Signing Key (2012) <cdimage@ubuntu.com>" not changed
gpg: Total number processed: 1
gpg:              unchanged: 1
gpg: Signature made Thu 27 Aug 2026 06:43:09 PM EDT
gpg:                using RSA key 843938DF228D22F7B3742BC0D94AA3F0EFE21092
gpg: Good signature from "Ubuntu CD Image Automatic Signing Key (2012) <cdimage@ubuntu.com>" [unknown]
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: 8439 38DF 228D 22F7 B374  2BC0 D94A A3F0 EFE2 1092
ubuntu-26.04.1-live-server-amd64.iso: OK
```

so I am expecting the iso to pass and when running `bin/check-isos ~/iso/ubuntu/24.04     # just a newly downloaded iso` I see that my download script worked perfectly and that my scan results match my download script results:

```bash
 bin/check-isos ~/iso/ubuntu/24.04     # just a newly downloaded iso

/home/initial/iso/ubuntu/24.04
  SIGNED      SHA256SUMS: SHA256SUMS.gpg good signature "Ubuntu CD Image Automatic Signing Key (2012) <cdimage@ubuntu.com>" [unknown]
  OK          ubuntu-24.04.5-live-server-amd64.iso

Summary: 1 OK, 0 failed, 0 unverified, 0 checksum files without a checked signature
```

So to check a  `debian/13` iso download the `SHA256SUMS` and `SHA256SUMS.sign` from the same Debian mirror folder as the ISO, and it will verify the iso image you downloaded.

### Updating or adding new ISOs to your ventoy usb stick

In my case I had an old ubuntu server iso `ubuntu-24.04.1-live-server-amd64.iso`and I am upgrading to the latest `ubuntu-24.04.5-live-server-amd64.iso`live server iso.

#### If your updating an iso delete the current version

```bash
# delete the old version on usb ventoy stick
rm /media/initial/Ventoy/iso/ubuntu/24.04/ubuntu-24.04.1-live-server-amd64.iso
```

#### If you are adding a ISO file, ensure for your target directory structure

```bash
mkdir -p /media/initial/Ventoy/iso/ubuntu/24.04/
mkdir -p /media/initial/Ventoy/iso/ubuntu/26.04/
mkdir -p /media/initial/Ventoy/iso/debian/13/
```

#### Copy over your ISO files and checksum info

```bash
cp ~/iso/debian/13/* /media/initial/Ventoy/iso/debian/13/.
cp ~/iso/ubuntu/24.04/* /media/initial/Ventoy/iso/ubuntu/24.04/.
cp ~/iso/ubuntu/26.04/* /media/initial/Ventoy/iso/ubuntu/26.04/.
```

### Notes:

You can run `check-isos` against your Ventoy iso directory but this may take significantly longer to run:

Example on an Ubuntu based debian system, your command may need to be different for running against your mount point, which can vary depending your workstations linux distribution and version.

```bash
bin/check-isos /media/initial/Ventoy/iso
```

## Requirements for building your Ventoy automated installations

### PyYAML and Jinja2

#### Installing to a Debian based system

If you want you can install them  system wide:

```bash
 sudo apt install python3-yaml python3-jinja2
```

#### Installing to a Python venv

Alternatively these to Python tools can be sand boxed by installing them by installing them to a Python virtual environment

### mkpasswd

See the document link below for installing mkpasswd on your linux distribution:

* [installing-mkpasswd](./en/docs/installing-mkpasswd)

## Optional

### Network Adapter and Cable

* You are probably going to want a physical connection to your network for speed as this is a net installer.
  * Network cable
  * Network dongle in the case of a laptop or for a physically isolated server that has no network card.

* Your system **may** be able to install from a wireless adapter but you will need to 



For development and troubleshooting work you may want additional tools.

### shellcheck

ShellCheck is a **free, open-source static analysis tool** written in **Haskell** that identifies syntax errors, semantic issues, and common pitfalls in **sh, bash, dash, and ksh** scripts.  It helps developers catch beginner mistakes, intermediate semantic problems, and advanced corner cases that might cause scripts to fail unexpectedly or behave counter-intuitively.

#### Installing to a Debian based system

```bash
apt install shellcheck
```

#### Installing to other linux based systems

See [Installing shellcheck](https://github.com/koalaman/shellcheck#installing)

## Preparation

### raw-node-input.yml

Before building your Ventoy automated installer you need to prepare the four variables that the build process requires for the out of box compatibility of the builder. Three more are optional: `ssh_port` (default 22), `ssh_sources` (default empty, meaning SSH is reachable from anywhere) and `password_batch` (the batch ID of the initial password). `raw-node-input.example.yml` explains them.

I edit  `raw-node-input.yml`directly and once everything is working as it should I copy it to `raw-node-input.example.yml` just in case it gets clobbered by some other developer or automation tool.

```bash
nano raw-node-input.yml
```

The `raw-node-input.yml` contains the fours variable values required to build your Ventoy automated installers.

I open raw-node-input.yml in my favourite quick editor

```bash
nano raw-node-input.yml
```

and then I generate my password hashed and copy paste into the open file, followed by the public key content and make any adjustments to the initial sudo users name.

### Creating  your users password hashes

The Ventoy auto installer sets a root password and an initial user's password. The two hashes come from different places.

**`root_password_hash`** is yours. Create it with this command and copy paste it into your `raw-node-input.yml` file:

```bash
mkpasswd --method=sha-512
```

**`initial_password_hash`** comes from the Ansible controller, which holds the initial password for the whole build batch (raw node contract 0.3.0). In uc-controller, `scripts/new_raw_batch.py` makes the password, keeps it, and prints a batch ID and a SHA-512 hash. Copy the hash into `initial_password_hash` and the batch ID into `password_batch`. uc-ventoy never sees the password itself.

`new_raw_batch.py` is not built yet. Until it is, make a batch by hand as described in [contracts.md](en/docs/contracts.md#the-initial-password): a password of at least 15 characters (the command there makes 24), kept in your password manager under a batch ID such as `raw-2026-11-17-manual`, and hashed with `mkpasswd --method=sha-512`.

The initial password works at the console and for `sudo`, not over SSH, which accepts your key only.

### Grab the contents of the public key you are using

```bash
cat ~/.ssh/my-public-key.pub
```

## Building and Testing

### If using a Python venv

if your using a venv, activate it now.

```bash
. ~/.venvs/ansible/current/bin/activate
```

### Run the build command

```bash
 bin/build
```

Output example:

```bash
warning: foomuuri not found, raw node firewall not checked (sudo apt install foomuuri)
Raw node firewall: SSH on port 22 from anywhere, not checked by nftables
warning: shellcheck not found, shared scripts not linted (sudo apt install shellcheck)
Report: raw-node-report prints valid YAML with the contract 0.3.0 fields (3 interfaces on this workstation), self-test OK
Targets:
  debian/13: preseed.cfg, hostname deb13-vt-xxxx, 7 files embedded, self-test OK
  ubuntu/24.04: user-data, hostname ub2404-vt-xxxx, 9 files embedded, self-test OK
  ubuntu/26.04: user-data, hostname ub2604-vt-xxxx, 9 files embedded, self-test OK

Wrote build/answer-files/ and build/ventoy/ventoy.json
```

#### foomuuri (optional)

A useful tool for developers, CI automations or editing the included firewall configuration.

Every raw node gets the same small Foomuuri firewall: SSH, DHCP and ping in, everything out. `bin/build` renders it once from `shared/foomuuri/00-raw-node.conf.j2`, using `ssh_port` and `ssh_sources` from your `raw-node-input.yml`, and embeds the same file in every answer file.

If Foomuuri is installed on your workstation (`sudo apt install foomuuri`), `bin/build` asks it to check the rendered file. Foomuuri hands the generated rules to `nft`, which talks to the kernel and therefore needs **sudo**. Without sudo the build only warns, as in the output example above. Run the full check once, and again whenever you change the firewall template, `ssh_port` or `ssh_sources`:

```bash
sudo bin/build --check-firewall-only
```

Output example when you are good to go:

```bash
Raw node firewall: SSH on port 22 from anywhere, foomuuri check OK
```

If you use a Python venv, sudo does not see it. Pass the venv's Python along:

```bash
sudo "$(command -v python3)" bin/build --check-firewall-only
```

An invalid firewall file would leave a new node without a firewall at its first boot, so this check is worth the minute it takes.

#### shellcheck (optional)

A useful tool for developers, CI automations or editing any included scripts

## Preview the changes

### Dry Run

Before you write the new configuration to your Ventoy stick, you want to do a dry run:

```bash
bin/build --stick /media/initial/Ventoy --dry-run
```

Complex output example:

```bash
warning: foomuuri check needs root for its nftables check; check the raw node firewall once with: sudo bin/build --check-firewall-only
Raw node firewall: SSH on port 22 from anywhere, not checked by nftables
Report: raw-node-report prints valid YAML with the contract 0.3.0 fields (3 interfaces on this workstation), self-test OK
Targets:
  debian/13: preseed.cfg, hostname deb13-vt-xxxx, 7 files embedded, self-test OK
  ubuntu/24.04: user-data, hostname ub2404-vt-xxxx, 9 files embedded, self-test OK
  ubuntu/26.04: user-data, hostname ub2604-vt-xxxx, 9 files embedded, self-test OK

Wrote build/answer-files/ and build/ventoy/ventoy.json

ISOs on /media/initial/Ventoy:
  /iso/debian/13: debian-13.7.0-amd64-netinst.iso
  /iso/ubuntu/24.04: ubuntu-24.04.5-live-server-amd64.iso
  /iso/ubuntu/26.04: ubuntu-26.04.1-live-server-amd64.iso

Stick: /media/initial/Ventoy
  would write answer-files/debian/13/preseed.cfg
  would write answer-files/ubuntu/24.04/user-data
  would write answer-files/ubuntu/26.04/user-data
  would create ventoy/ventoy.json: 3 raw node entries, 0 other auto_install entries kept
```

#### Dry run output notes:

So you can see that it again reminds us of the `foomuuri` and `shellcheck` options for developers and strict CI and automation integrations.

The other item of note is that it has identified a second ventoy.json file

```bash
bin/build --stick /media/initial/Ventoy --dry-run
```

### Writing custom answer files to your Ventoy usb stick

#### Requires

Requires Python 3 with PyYAML and Jinja2 on your workstations

#### Optional Check

When they are installed, optional checks will use:

*  debconf-set-selections
* foomuuri
* shellcheck and:
* gpg

#### Writing your to your Ventoy stick

 Once you have done your dry run and it includes the requirements for any options checks you need, copy your custom installers over to your usb Ventoy stick

```bash
bin/build --stick /media/initial/Ventoy
```

Output example:

```bash
warning: foomuuri not found, raw node firewall not checked (sudo apt install foomuuri)
Raw node firewall: SSH on port 22 from anywhere, not checked by nftables
warning: shellcheck not found, shared scripts not linted (sudo apt install shellcheck)
Report: raw-node-report prints valid YAML with the contract 0.3.0 fields (3 interfaces on this workstation), self-test OK
Targets:
  debian/13: preseed.cfg, hostname deb13-vt-xxxx, 7 files embedded, self-test OK
  ubuntu/24.04: user-data, hostname ub2404-vt-xxxx, 9 files embedded, self-test OK
  ubuntu/26.04: user-data, hostname ub2604-vt-xxxx, 9 files embedded, self-test OK

Wrote build/answer-files/ and build/ventoy/ventoy.json

ISOs on /media/initial/Ventoy:
  /iso/debian/13: debian-13.7.0-amd64-netinst.iso
  /iso/ubuntu/24.04: ubuntu-24.04.5-live-server-amd64.iso
  /iso/ubuntu/26.04: ubuntu-26.04-live-server-amd64.iso

Stick: /media/initial/Ventoy
  writing answer-files/debian/13/preseed.cfg
  writing answer-files/ubuntu/24.04/user-data
  writing answer-files/ubuntu/26.04/user-data
  updating ventoy/ventoy.json: 3 raw node entries, 0 other auto_install entries kept
  backup: ventoy/ventoy.json.raw-node-backup-20261115-011245
  written to the device and read back: OK
warning: /media/initial/Ventoy/answer-files/ventoy.json is ignored by Ventoy, which reads only ventoy/ventoy.json; consider removing it
```

In the output at the end you will notice the warning about a second ventoy.json that I have stored here:

```bash
ventoy/ventoy.json
```

It's an extra that I used when developing, I should clean it up or perhaps place it in a backup directory so I can use it as a "known good" in future development. I will leave it alone for the time being.

## Testing against real systems

Now I am ready to test my automated configuration against real hardware!!!

On my local system

* I will get out of all directories on my USB stick so that I can saftly remove it.

On my test system

* Backup anything I want to save

* Ensure that I have a physical network (cable) connection to my testing system
  * hardwiringwifi may work but since this is a new installer I what thing to go as quickly as possible so I am hard wiring via a network cable

* Insert the Ventoy installer into a bootable USB port on the target system
* Boot the system to the usb drive
  * On my old Sansung laptop I need to press F9 right after starting it. If you are not sure, check your hardware manual or the internet for instructions on booting to a usb stick on your hardwate.

* Partitioning is not automated so for the Debian 13 installation I chose the LVM option and not (LUKS?) encryption as that requires a lot more work for allowing Ansible to complete my full configuration.
* 

## Layout

- `raw-node-input.yml`: your four values, the two optional SSH values and the optional password batch ID (git-ignored).
- `shared/`: files written identically on every node: SSH policy, the Foomuuri firewall template, `raw-node-report`, `raw-node-firstboot` and its systemd unit, and a cloud-init setting and a systemd drop-in used on Ubuntu.
- `<distro>/<version>/*.j2`: one answer file template per target. A new folder is a new target.
- `bin/build`: renders every target, embeds the shared files, re-extracts them to prove they arrive byte for byte, and writes `build/`.
- `build/`: mirrors the stick root: `answer-files/<distro>/<version>/...` and `ventoy/ventoy.json` (the `auto_install` section only).
- `confirmations/`: the confirmation files collected from nodes.
- `en/docs/`: the contract coverage, the baseline and the target documents.

## On the stick

ISOs stay where they are, under `/iso/<distro>/<version>/`. Ventoy matches each folder rather than each ISO, so a new point release only needs the ISO swapped. `bin/build --stick` copies the answer files to `/answer-files/<distro>/<version>/`, verifies ISOs against `SHA256SUMS` where present, backs up `ventoy/ventoy.json`, and replaces only the raw node `auto_install` entries, keeping all other settings.

## On the node

At first boot, `raw-node-firstboot.service` appends the MAC suffix to the hostname (`deb13-vt` becomes `deb13-vt-a1b2`), writes `/var/lib/raw-node/confirmation.yml` in the format of the raw node contract, and shows it at the console login prompt. `sudo raw-node-report` prints it again at any time.

SSH accepts the key only. The initial user's batch password works at the console, which is the recovery path while the node is raw, and `sudo` asks for it.

## History

Running the following command will give you an older working version of this project that works using nftables. Tested on Debian/13, configurations for Ubuntu 24.04 and 26.04 are likly to work as well but untested.

```bash
git worktree add ../uc-ventoy-baseline-0.2.0 baseline-0.2.0
```

