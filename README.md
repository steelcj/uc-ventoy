# ventoy-raw-node

Generator for Ventoy answer files that build a raw node: the minimal, identical starting state of every server before Ansible takes over. See `docs/ventoy-raw-node-baseline-v0.2.0.md` for the definition.

Run it on your workstation. Only its output goes on the Ventoy stick.

## Usage

Use this to quickly generate new automated configurations for multiple linux versions quickly and efficiently.

## Recommendations

I would highly recommend running the included utility against any ISO's that you will add your Ventoy or manually verifying their checksums before using them for installations.

### bin/check-isos

bin/check-isos is used on your local system to check a directory of one or more iso images on your local system.

**`bin/check-isos`** takes any directory and rehashes every ISO on every run:

```sh
bin/check-isos                        # everything under ~/iso
bin/check-isos ~/iso/ubuntu/24.04     # just a newly downloaded iso
```

Each ISO is reported as `OK`, `FAILED` or `UNVERIFIED`, and the tool exits with an error if any ISO fails.

- It catches a corrupted ISO and a checksum file edited after signing.
- It listed ISOs with no checksum to check against

For example, to check a  `debian/13` iso download the `SHA256SUMS` and `SHA256SUMS.sign` from the same Debian mirror folder as the ISO, and it will verify the iso image you downloaded.

## Requirements

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

* [installing-mkpasswd](./docs/installing-mkpasswd)

## Optional tools

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

Before building your Ventoy automated installer you need to prepare the four variables that the build process requires for the out of box compatibility of the builder.

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

By default the version of the Ventoy auto installer supports a root password and an initial users password.

Create your hashes with this command and copy paste into your `raw-node-input.yml`file:

```bash
mkpasswd --method=sha-512
```

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
warning: nft -c needs root; check the ruleset once with: sudo nft -c -f shared/nftables/nftables.conf
warning: shellcheck not found, shared scripts not linted (sudo apt install shellcheck)
Targets:
  debian/13: preseed.cfg, hostname deb13-vt-xxxx, 6 files embedded, self-test OK
  ubuntu/24.04: user-data, hostname ub2404-vt-xxxx, 7 files embedded, self-test OK
  ubuntu/26.04: user-data, hostname ub2604-vt-xxxx, 7 files embedded, self-test OK

Wrote build/answer-files/ and build/ventoy/ventoy.json
```

#### nftables

This setup is designed to work with as many linux distro's as possible to it ships with support for an nftables firewall. If you are modifying the default firewall configuration (port 22 only at this point in time) the you are going to want to ensure that your firewall configuration is valid. You do that with a utility called `nft`

If you have a look at the output example above you will see the command you want to run in order to do this. Run it now if you want to confirm that the out of the box firewall configuration in this project is valid.

The `nft` command requires **sudo** (or root privileges) because it interacts directly with the **Linux kernel** via the **libnftnl API** to manage the firewall's packet filtering rules.  Access from userland to the kernel is restricted to prevent unauthorized modification of network security policies.

In other words you have to use sudo for this one my friend!

```bash
sudo nft -c -f shared/nftables/nftables.conf
echo $?
```

Output example when you are good to go:

```bash
0
```

## Preview the changes

### Dry Run

Before you write the new configuration to your Ventoy stick, you want to do a dry run:

```bash
bin/build --stick /media/initial/Ventoy --dry-run
```

Complex output example:

```bash
warning: nft -c needs root; check the ruleset once with: sudo nft -c -f shared/nftables/nftables.conf
warning: shellcheck not found, shared scripts not linted (sudo apt install shellcheck)
Targets:
  debian/13: preseed.cfg, hostname deb13-vt-xxxx, 6 files embedded, self-test OK
  ubuntu/24.04: user-data, hostname ub2404-vt-xxxx, 7 files embedded, self-test OK
  ubuntu/26.04: user-data, hostname ub2604-vt-xxxx, 7 files embedded, self-test OK

Wrote build/answer-files/ and build/ventoy/ventoy.json

ISOs on /media/initial/Ventoy:
  /iso/debian/13: debian-13.7.0-amd64-netinst.iso
  /iso/ubuntu/24.04: ubuntu-24.04.2-live-server-amd64.iso
  /iso/ubuntu/26.04: ubuntu-26.04-live-server-amd64.iso

Stick: /media/initial/Ventoy
  would write answer-files/debian/13/preseed.cfg
  would write answer-files/ubuntu/24.04/user-data
  would write answer-files/ubuntu/26.04/user-data
  would update ventoy/ventoy.json: 3 raw node entries, 0 other auto_install entries kept
warning: /media/initial/Ventoy/answer-files/ventoy.json is ignored by Ventoy, which reads only ventoy/ventoy.json; consider removing it
```

#### Dry run output notes:

So you can see that it again reminds us of the `nft`and `spellcheck`options for developers and strict CI and automation integrations.

The other item of note is that it has identified a second ventoy.json file



1. : `bin/build --stick /media/initial/Ventoy --dry-run`
2. Write to the stick: `bin/build --stick /media/initial/Ventoy`

Requires Python 3 with PyYAML and Jinja2 (`sudo apt install python3-yaml python3-jinja2`). Optional checks use `debconf-set-selections`, `nft`, `shellcheck` and `gpg` when they are installed.

## Layout

- `raw-node-input.yml`: your four values (git-ignored).
- `shared/`: files written identically on every node: SSH policy, bootstrap firewall, `raw-node-report`, `raw-node-firstboot` and its systemd unit, and a cloud-init setting used on Ubuntu.
- `<distro>/<version>/*.j2`: one answer file template per target. A new folder is a new target.
- `bin/build`: renders every target, embeds the shared files, re-extracts them to prove they arrive byte for byte, and writes `build/`.
- `build/`: mirrors the stick root: `answer-files/<distro>/<version>/...` and `ventoy/ventoy.json` (the `auto_install` section only).
- `confirmations/`: the confirmation files collected from nodes.
- `docs/`: the baseline and target documents.

## On the stick

ISOs stay where they are, under `/iso/<distro>/<version>/`. Ventoy matches each folder rather than each ISO, so a new point release only needs the ISO swapped. `bin/build --stick` copies the answer files to `/answer-files/<distro>/<version>/`, verifies ISOs against `SHA256SUMS` where present, backs up `ventoy/ventoy.json`, and replaces only the raw node `auto_install` entries, keeping all other settings.

## On the node

At first boot, `raw-node-firstboot.service` appends the MAC suffix to the hostname (`deb13-vt` becomes `deb13-vt-a1b2`), writes `/var/lib/raw-node/confirmation.yml`, and shows it at the console login prompt. `sudo raw-node-report` prints it again at any time.
