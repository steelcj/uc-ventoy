# Ventoy Automated Configuration for Debian and Ubuntu that produce the same baseline

- Debian Stable: Debian Installer with preseed.
- Ubuntu Server LTS: Subiquity with cloud-init autoinstall.
- Both: root account with a local password, `initial` with sudo, your SSH public key, SSH password authentication, and an `nftables` firewall allowing inbound SSH only.
- Both: leave disk partitioning for manual confirmation until we have a reliable disk identifier.

The important distinction is that **the firewall allows SSH traffic; SSH then decides whether the connection authenticates with a key or a password.**

## **1. Ventoy directory structure**

On the Ventoy data partition:

```
/
├── ISO/
│   ├── debian/
│   │   └── debian-stable-amd64.iso
│   └── ubuntu/
│       └── ubuntu-server-lts-amd64.iso
│
└── ventoy/
    ├── ventoy.json
    └── answer-files/
        ├── debian-preseed.cfg
        └── ubuntu-user-data
```

The ISO filenames above are placeholders. Use the actual filenames you download, or rename the files to match.

## **2. Ventoy configuration**

```
/ventoy/ventoy.json
{
  "auto_install": [
    {
      "image": "/ISO/debian/debian-stable-amd64.iso",
      "template": "/ventoy/answer-files/debian-preseed.cfg"
    },
    {
      "image": "/ISO/ubuntu/ubuntu-server-lts-amd64.iso",
      "template": "/ventoy/answer-files/ubuntu-user-data"
    }
  ]
}
```

I have deliberately omitted `autosel`. Ventoy will present the installation-template selection rather than silently selecting one. This is useful protection against accidentally starting an unattended installation. 

ventoy.net



## **3. Shared security configuration**

Both installations will use the same SSH configuration.

```
/etc/ssh/sshd_config.d/10-local-auth.conf
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication yes
KbdInteractiveAuthentication no
UsePAM yes
```

This gives us:

- Local root account with a password.
- No direct SSH login as root.
- SSH key authentication for `initial`.
- SSH password authentication for `initial`.

The firewall configuration will also be identical.

```
/etc/nftables.conf
#!/usr/sbin/nft -f

flush ruleset

table inet filter {
    chain input {
        type filter hook input priority filter;
        policy drop;

        iifname "lo" accept
        ct state invalid drop
        ct state established,related accept

        ip protocol icmp accept
        meta l4proto ipv6-icmp accept

        tcp dport 22 ct state new accept
    }

    chain forward {
        type filter hook forward priority filter;
        policy drop;
    }

    chain output {
        type filter hook output priority filter;
        policy accept;
    }
}
```

This permits loopback, established connections, ICMP/ICMPv6, and new SSH connections. Other unsolicited inbound traffic is dropped.

## **4. Debian preseed**

```
/ventoy/answer-files/debian-preseed.cfg
### Locale and keyboard

d-i debian-installer/locale string en_CA.UTF-8
d-i keyboard-configuration/xkb-keymap select us
d-i time/zone string America/Toronto
d-i clock-setup/utc boolean true
d-i clock-setup/ntp boolean true

### Network

d-i netcfg/get_hostname string debian-server
d-i netcfg/get_domain string local

### Installer

d-i debconf/priority select critical

### Root account

d-i passwd/root-login boolean true
d-i passwd/root-password-crypted password ROOT_PASSWORD_HASH

### Initial user

d-i passwd/make-user boolean true
d-i passwd/user-fullname string Initial User
d-i passwd/username string initial
d-i passwd/user-password-crypted password INITIAL_PASSWORD_HASH

### Minimal package selection

tasksel tasksel/first multiselect standard, ssh-server

d-i pkgsel/include string sudo nftables
d-i pkgsel/upgrade select none
d-i pkgsel/update-policy select none

### Bootloader

d-i grub-installer/only_debian boolean true

### Post-install configuration

d-i preseed/late_command string \
    in-target usermod -aG sudo initial; \
    mkdir -p /target/home/initial/.ssh; \
    chmod 0700 /target/home/initial/.ssh; \
    printf '%s\n' 'YOUR_SSH_PUBLIC_KEY' > /target/home/initial/.ssh/authorized_keys; \
    in-target chown -R initial:initial /home/initial/.ssh; \
    in-target chmod 0600 /home/initial/.ssh/authorized_keys; \
    mkdir -p /target/etc/ssh/sshd_config.d; \
    printf '%s' 'UGVybWl0Um9vdExvZ2luIG5vClB1YmtleUF1dGhlbnRpY2F0aW9uIHllcwpQYXNzd29yZEF1dGhlbnRpY2F0aW9uIHllcwpLYmRJbnRlcmFjdGl2ZUF1dGG9uIG5vClVzZVBBTSB5ZXMK' | base64 -d > /target/etc/ssh/sshd_config.d/10-local-auth.conf; \
    printf '%s' 'IyEvdXNyL3NiaW4vbmZ0IC1mCmZsdXNoIHJ1bGVzZXQKCnRhYmxlIGluZXQgZmlsdGVyIHsKICAgIGNoYWluIGlucHV0IHsKICAgICAgICB0eXBlIGZpbHRlciBob29rIGlucHV0IHByaW9yaXR5IGZpbHRlcjsKICAgICAgICBwb2xpY3kgZHJvcDsKCiAgICAgICAgaWlmbmFtZSAibG8iIGFjY2VwdAogICAgICAgIGN0IHN0YXRlIGludmFsaWQgZHJvcAogICAgICAgIGN0IHN0YXRlIGVzdGFibGlzaGVkLHJlbGF0ZWQgYWNjZXB0CgogICAgICAgIGlwIHByb3RvY29sIGljbXAgYWNjZXB0CiAgICAgICAgbWV0YSBsNHByb3RvIGlwdjYtaWNtcCBhY2NlcHQKCiAgICAgICAgdGNwIGRwb3J0IDIyIGN0IHN0YXRlIG5ldyBhY2NlcHQKICAgIH0KCiAgICBjaGFpbiBmb3J3YXJkIHsKICAgICAgICB0eXBlIGZpbHRlciBob29rIGZvcndhcmQgcHJpb3JpdHkgZmlsdGVyOwogICAgICAgIHBvbGljeSBkcm9wOwogICAgfQoKICAgIGNoYWluIG91dHB1dCB7CiAgICAgICAgdHlwZSBmaWx0ZXIgaG9vayBvdXRwdXQgcHJpb3JpdHkgZmlsdGVyOwogICAgICAgIHBvbGljeSBhY2NlcHQ7CiAgICB9Cn0K' | base64 -d > /target/etc/nftables.conf; \
    in-target systemctl enable ssh.service nftables.service
```

**Important:** I would not use that late-command block unchanged. The base64 SSH configuration above should be generated from the shared file rather than copied manually, and the long base64 firewall payload is difficult to maintain. I have shown the mechanism, but I would replace these embedded payloads with a generated, shared configuration script in the finished version.

The significant point is that Debian's `late_command` can configure the installed system under `/target` before reboot. 

debian.org+1



## **5. Ubuntu autoinstall**

```
/ventoy/answer-files/ubuntu-user-data
#cloud-config

autoinstall:
  version: 1

  locale: en_CA.UTF-8

  keyboard:
    layout: us

  timezone: America/Toronto

  hostname: ubuntu-server

  # Keep storage selection interactive.
  interactive-sections:
    - storage

  identity:
    hostname: ubuntu-server
    username: initial
    realname: Initial User
    password: "INITIAL_PASSWORD_HASH"
    groups:
      - adm
      - sudo

  ssh:
    install-server: true
    allow-pw: true
    authorized-keys:
      - "YOUR_SSH_PUBLIC_KEY"

  packages:
    - sudo
    - nftables

  user-data:
    users:
      - default
      - name: root
        passwd: "ROOT_PASSWORD_HASH"
        lock_passwd: false
        shell: /bin/bash

  late-commands:
    - mkdir -p /target/home/initial/.ssh
    - chmod 0700 /target/home/initial/.ssh
    - chown -R 1000:1000 /target/home/initial/.ssh
    - curtin in-target --target=/target -- sh -c 'printf "%s\n" "YOUR_SSH_PUBLIC_KEY" > /home/initial/.ssh/authorized_keys'
    - curtin in-target --target=/target -- chmod 0600 /home/initial/.ssh/authorized_keys
    - mkdir -p /target/etc/ssh/sshd_config.d
    - curtin in-target --target=/target -- systemctl enable ssh.service nftables.service
```

This is the Ubuntu-specific starting point. The SSH and nftables configuration files from section 3 still need to be written into `/target` in the Ubuntu `late-commands`, just as in Debian.

Ubuntu's autoinstall reference documents the identity, groups, SSH keys, password authentication, and late-command mechanisms. 

GitHub+1



## **6. Generate password hashes**

Install `whois` on your workstation if necessary:

```
sudo apt install whois
```

Generate a SHA-512 password hash:

```
mkpasswd --method=sha-512
```

Generate separate hashes for:

- `initial`
- `root`

Replace the two hash placeholders in both installation configurations.

Do not put plaintext passwords in the Ventoy files. Remember that the hashes and public key are still sensitive installation material, so protect the USB.

## **7. Validation after installation**

Once installed, check the following:

```
sudo -v
sudo id

sudo sshd -t

sudo sshd -T | grep -E \
  'permitrootlogin|passwordauthentication|pubkeyauthentication'

sudo nft list ruleset

sudo systemctl status ssh
sudo systemctl status nftables
```

Test both SSH authentication methods from another machine:

```
# Public key
ssh -o PreferredAuthentications=publickey initial@SERVER_IP

# Password
ssh -o PreferredAuthentications=password \
    -o PubkeyAuthentication=no initial@SERVER_IP
```

Also verify that root cannot log in over SSH.

## **8. The implementation I would actually maintain**

I would avoid maintaining two copies of the SSH and firewall configurations. Instead, keep the shared files in a small repository and generate the installer-specific answer files.

```
server-install/
├── README.md
├── shared/
│   ├── ssh/
│   │   └── 10-local-auth.conf
│   └── nftables/
│       └── nftables.conf
├── debian/
│   └── preseed.cfg.in
├── ubuntu/
│   └── user-data.in
├── ventoy/
│   └── ventoy.json
└── bin/
    └── build-install-media.py
```

The Python build script would:

1. Read the shared SSH and nftables configuration.
2. Insert them into the Debian and Ubuntu installer configurations.
3. Insert the public key and password hashes from protected local inputs.
4. Generate the final files for the Ventoy USB.
5. Validate the generated YAML and JSON.
6. Never write private keys or plaintext passwords into the output.

This also avoids the maintenance problem of hand-editing long embedded shell commands.

**One qualification:** these are configuration templates, not yet a tested turnkey installer. I would validate both in Incus virtual machines before using them on physical hardware, especially the account creation, root password, SSH configuration, and firewall activation. Disk partitioning should remain a deliberate decision until we specify how to identify the target device.

For the next iteration, I would build the Python generator and make the two answer files genuinely share the same SSH and firewall source files. That gives you one security baseline rather than two configurations that can gradually drift apart.