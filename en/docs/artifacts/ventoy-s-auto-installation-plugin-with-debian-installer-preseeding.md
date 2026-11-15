# Ventoy's Auto Installation plugin with Debian Installer preseeding

I would use **Ventoy's Auto Installation plugin with Debian Installer preseeding (**`preseed.cfg`**)**.

This is a particularly good fit because you don't need to rebuild or modify the Debian ISO. Ventoy supplies the preseed configuration when the selected ISO boots. Debian Installer then uses it to configure the operating system.

You can automate almost everything, including account creation, SSH configuration, package selection, and firewall configuration. 

ventoy.net+1



## **1. Proposed installation architecture**

**Ventoy USB**

Bootable installation media

**Debian Stable ISO**

Original, unmodified amd64 installation image

**Ventoy Auto Installation**

ventoy.json associates the ISO with its preseed template

**Debian Installer / preseed.cfg**

Automated installation and system configuration

Target system

**Minimal Debian server**

- Root account enabled
- `initial` account with sudo
- SSH public key installed
- SSH password authentication enabled
- nftables firewall
- No desktop environment

## **2. Ventoy directory structure**

I'd organize the Ventoy data partition as follows:

```
Ventoy/
├── ISO/
│   └── debian/
│       └── debian-stable-amd64.iso
│
└── ventoy/
    ├── ventoy.json
    └── scripts/
        └── debian-server-preseed.cfg
```

This separates installation media from installation configuration.

The same USB device could eventually contain multiple Debian installation profiles, for example:

- Minimal server
- Development server
- Desktop workstation
- Virtualization host

Each profile could have its own preseed configuration.

## **3. Configure Ventoy**

Create `/ventoy/ventoy.json`:

```
{
  "auto_install": [
    {
      "image": "/ISO/debian/debian-stable-amd64.iso",
      "template": "/ventoy/scripts/debian-server-preseed.cfg",
      "autosel": 1
    }
  ]
}
```

The `autosel` value selects the first installation template.

I would initially leave the installation-template selection interactive rather than silently selecting it. Ventoy supports this configuration and allows you to change templates without modifying the ISO. 

ventoy.net



## **4. Debian preseed configuration**

The preseed file controls Debian Installer.

I'd divide the configuration into five areas:

1. Locale and network.
2. Root and administrative accounts.
3. Package selection.
4. SSH configuration.
5. Firewall configuration.

### **4.1 Locale and installation defaults**

```
d-i debian-installer/locale string en_CA.UTF-8
d-i keyboard-configuration/xkb-keymap select us
d-i time/zone string America/Toronto

d-i clock-setup/utc boolean true
d-i clock-setup/ntp boolean true

d-i debconf/priority select critical
```

Adjust the keyboard layout to match the target hardware.

### **4.2 Root account**

I would enable the root account with a password hash rather than putting a plaintext password in the preseed file.

```
d-i passwd/root-login boolean true
d-i passwd/root-password-crypted password $6$REPLACE_WITH_ROOT_HASH
```

Generate a SHA-512 password hash on your workstation:

```
sudo apt install whois
mkpasswd --method=sha-512
```

Use a strong, unique password.

The root account will be available at the local console. I would nevertheless disable direct root SSH login.

### **4.3 Create the** `initial` **user**

```
d-i passwd/make-user boolean true
d-i passwd/user-fullname string Initial User
d-i passwd/username string initial
d-i passwd/user-password-crypted password $6$REPLACE_WITH_INITIAL_HASH

d-i user-setup/allow-password-weak boolean false
d-i user-setup/encrypt-home boolean false
```

The installer creates the user. We subsequently add the account to the `sudo` group.

### **4.4 Install required packages**

```
tasksel tasksel/first multiselect standard, ssh-server

d-i pkgsel/include string sudo openssh-server nftables

d-i pkgsel/upgrade select none
d-i pkgsel/update-policy select none
```

This installs:

| Package          | Purpose                             |
| ---------------- | ----------------------------------- |
| `sudo`           | Administrative privilege escalation |
| `openssh-server` | Remote administration               |
| `nftables`       | Host firewall                       |

The standard system utilities task is retained.

**Offline installation consideration:** CD-1 does not guarantee that every optional package is present. If you require all these packages offline, use DVD-1 or ensure the installation has access to a Debian package mirror.

## **5. Install the SSH public key**

This is one of the more useful parts of automating the installation.

The public key should be installed into:

```
/home/initial/.ssh/authorized_keys
```

Use your actual public key, for example:

```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... initial@workstation
```

The public key is not secret, so including it in the preseed configuration is acceptable. Never include your private key.

Add the following to the preseed `late_command`:

```
d-i preseed/late_command string \
    in-target usermod -aG sudo initial; \
    in-target install -d -m 0700 -o initial -g initial /home/initial/.ssh; \
    in-target sh -c 'printf "%s\n" "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... initial@workstation" > /home/initial/.ssh/authorized_keys'; \
    in-target chown initial:initial /home/initial/.ssh/authorized_keys; \
    in-target chmod 0600 /home/initial/.ssh/authorized_keys
```

Replace the example key with your actual public key.

This creates the `.ssh` directory, installs the key, and establishes the required ownership and permissions.

## **6. Configure SSH authentication**

I would explicitly configure SSH rather than relying on distribution defaults.

Create:

```
/etc/ssh/sshd_config.d/10-local-auth.conf
```

With:

```
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication yes
```

This gives you:

- Root account enabled locally.
- Root SSH login disabled.
- Public-key SSH authentication enabled.
- Password SSH authentication enabled for regular accounts.

Both authentication methods are available to `initial`.

The public key is preferable for normal administration, while password authentication provides an alternative recovery path.

## **7. Configure the firewall**

I'd use native `nftables` rather than adding another firewall abstraction.

The desired policy:

| Traffic                 | Policy |
| ----------------------- | ------ |
| Loopback                | Allow  |
| Established connections | Allow  |
| SSH TCP/22              | Allow  |
| IPv4 ICMP               | Allow  |
| IPv6 ICMP               | Allow  |
| Other incoming traffic  | Deny   |
| Forwarded traffic       | Deny   |
| Outgoing traffic        | Allow  |

The ICMP exceptions are intentional: they support diagnostics and essential IPv6 network functions. They do not open additional application services.

Create `/etc/nftables.conf`:

```
#!/usr/sbin/nft -f

flush ruleset

table inet filter {

    chain input {
        type filter hook input priority filter;
        policy drop;

        iifname "lo" accept

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

Enable the firewall:

```
systemctl enable nftables
systemctl enable ssh
```

The firewall will load its rules on subsequent boots.

**Important:** Validate the ruleset before applying it remotely. An incorrect firewall configuration can lock you out of a server.

## **8. Complete the installation**

At this stage, Debian Installer will:

1. Partition and format the target disk.
2. Install Debian.
3. Configure the root account.
4. Create `initial`.
5. Install the selected packages.
6. Add `initial` to the sudo group.
7. Install the SSH public key.
8. Configure SSH authentication.
9. Configure and enable the firewall.
10. Install GRUB.
11. Reboot.

I would leave disk partitioning as a manual confirmation initially.

Once the installation process has been tested, disk partitioning can also be automated using a stable `/dev/disk/by-id/` identifier.

I would not automatically select the largest disk: that is an unnecessary risk when a Ventoy USB and multiple storage devices may be connected. Ventoy itself warns about incorrect disk selection in unattended installations. 

ventoy.net



## **9. Putting the SSH and firewall configuration into the installer**

The configuration snippets above need to be written into the installed system. I would extend `late_command` to do that.

For example, the SSH portion can be created with:

```
in-target sh -c \
  'printf "%s\n" \
  "PermitRootLogin no" \
  "PubkeyAuthentication yes" \
  "PasswordAuthentication yes" \
  > /etc/ssh/sshd_config.d/10-local-auth.conf'
```

Similarly, the firewall configuration can be written using `printf` or a separate configuration file copied into the target system.

Then enable both services:

```
in-target systemctl enable ssh
in-target systemctl enable nftables
```

I would keep these operations in the installer's late-command stage so that the installed system is configured before its first normal boot.

## **10. First-boot verification**

After installation, connect using the `initial` account.

Verify sudo:

```
sudo -v
sudo id
```

Verify the SSH configuration:

```
sudo sshd -t
sudo sshd -T | grep -E \
  'permitrootlogin|passwordauthentication|pubkeyauthentication'
```

Expected:

```
permitrootlogin no
pubkeyauthentication yes
passwordauthentication yes
```

Verify the firewall:

```
sudo nft list ruleset
sudo systemctl status nftables
```

Verify SSH:

```
sudo systemctl status ssh
```

From another machine, test key authentication:

```
ssh -o PreferredAuthentications=publickey initial@SERVER_IP
```

Then test password authentication:

```
ssh -o PreferredAuthentications=password \
    -o PubkeyAuthentication=no initial@SERVER_IP
```

Finally, verify that other incoming ports are blocked.

## **11. A further improvement: separate installation from configuration**

For a repeatable server deployment, I would actually separate the work into two stages.

Stage 1

**Debian Installer (preseed)**

- Partition and install Debian
- Create root
- Create `initial`
- Set passwords
- Install SSH
- Install public key
- Enable sudo
- Install and configure firewall

Stage 2

**Ansible provisioning**

- Establish baseline configuration
- Configure locale and timezone
- Configure SSH hardening
- Configure firewall policy
- Apply security updates
- Configure monitoring and logging
- Apply server-specific roles

This gives you a consistent installation process while retaining the flexibility to change server configurations independently.

It also means the preseed file does not become a second configuration-management system.

## **12. Security considerations**

A few things I would be particularly careful about:

1. **Password hashes:** Never put plaintext passwords in the Ventoy configuration.
2. **Public keys:** Public keys are safe to distribute, but protect the USB because it contains your installation configuration.
3. **Root access:** Keep root enabled for console recovery but disable root SSH access.
4. **Firewall:** Default-deny incoming traffic; explicitly allow SSH.
5. **Disk selection:** Require confirmation until the target disk selection has been thoroughly tested.
6. **SSH passwords:** Password authentication is convenient, but use strong passwords and consider disabling it once key-based access and recovery procedures have been tested.
7. **Network exposure:** Restrict SSH to trusted networks where possible.

## **References**

- [Ventoy Auto Installation Plugin](https://www.ventoy.net/en/plugin_autoinstall.html?utm_source=chatgpt.com)
- [Ventoy Configuration](https://www.ventoy.net/en/plugin_autoinstall.html?utm_source=chatgpt.com)
- [Debian Installer: Automating Installation Using Preseeding](https://www.debian.org/releases/stable/amd64/apbs02.en.html?utm_source=chatgpt.com)
- [Debian Installer Preseed Examples](https://www.debian.org/releases/trixie/example-preseed.txt?utm_source=chatgpt.com)
- [Debian OpenSSH Server](https://wiki.debian.org/SSH?utm_source=chatgpt.com)
- [nftables Documentation](https://wiki.nftables.org/?utm_source=chatgpt.com)

## **My recommendation**

I would build this as a small, version-controlled installation project rather than maintain a collection of manually edited files on the USB.

For example:

```
debian-install/
├── README.md
├── ventoy/
│   └── ventoy.json
├── preseed/
│   └── debian-server.cfg
├── files/
│   ├── ssh/
│   │   └── 10-local-auth.conf
│   └── nftables/
│       └── nftables.conf
└── tests/
    └── verify-install.sh
```

Keep the public SSH key and password hashes in a locally protected configuration, rather than committing them to a public repository.

**One important design decision remains:** I would keep the installation generic and let Ansible own the long-term server configuration. The installer should establish a secure, accessible, minimally functional Debian system. Everything beyond that belongs in provisioning.

That gives you a reusable Ventoy installation medium for multiple machines without coupling the installation process to any particular server role.