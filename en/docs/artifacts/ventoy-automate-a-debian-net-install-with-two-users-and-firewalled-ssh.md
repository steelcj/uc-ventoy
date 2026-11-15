# Ventoy automate a debian net install with two users

I would use **Debian Installer preseed + Ventoy Auto Install**. This is a good fit for a Debian netinst ISO: the installer downloads the required packages, configures the accounts and services, and applies the initial firewall before the first reboot.

The resulting system would have:

- `root`: enabled, with a locally usable password; SSH root login disabled.
- `initial`: regular user, password set, member of `sudo`.
- OpenSSH server installed.
- Your public SSH key installed in `/home/initial/.ssh/authorized_keys`.
- SSH password authentication enabled as a fallback.
- `nftables` enabled, default-deny inbound, TCP port 22 permitted.
- Outbound connections permitted, so package updates and normal networking work.

One distinction: the firewall allows inbound TCP port 22. SSH itself supports both public-key and password authentication.

## **1. Ventoy layout**

Assuming your Ventoy USB has the usual data partition:

```shell
/
├── ISO/
│   └── debian/
│       └── 13
│           └── debian-13.7.0-amd64-netinst.iso
│
└── ventoy/
    ├── ventoy.json
    └── debian/
        └── 13
            └── preseed.cfg
```

## **2. Configure Ventoy**

`/ventoy/ventoy.json`:

```json
{
  "auto_install": [
    {
      "image": "/ISO/debian/13/debian-13.7.0-amd64-netinst.iso",
      "template": "/ventoy/debian/13/preseed.cfg"
    }
  ]
}
```

I would omit `autosel` initially. That leaves the template selection available for confirmation rather than silently starting the installation.

## Generate password hashes

On your workstation:

```
sudo apt install whois
```

Generate each hash separately:

```
mkpasswd --method=sha-512
```

Use the resulting hashes in the preseed file. Do not put plaintext passwords in it.

## 

## Debian preseed configuration

Replace these three placeholders:

- `ROOT_PASSWORD_HASH`
- `INITIAL_PASSWORD_HASH`
- `YOUR_SSH_PUBLIC_KEY`

The firewall allows inbound SSH and ICMP/ICMPv6 control traffic, but no other inbound application services. It permits outbound connections. If you want to block even ICMP, that can be tightened, but I generally recommend retaining it for network diagnostics and IPv6 operation.

Create `/ventoy/debian/preseed.cfg`:

```
#_preseed_V1

### Locale and keyboard

d-i debian-installer/locale string en_CA.UTF-8
d-i keyboard-configuration/xkb-keymap select us
d-i time/zone string America/Toronto
d-i clock-setup/utc boolean true
d-i clock-setup/ntp boolean true

### Network

d-i netcfg/get_hostname string debian-server
d-i netcfg/get_domain string local
d-i netcfg/choose_interface select auto

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

### Package selection

tasksel tasksel/first multiselect standard, ssh-server

d-i pkgsel/include string sudo nftables
d-i pkgsel/upgrade select none
d-i pkgsel/update-policy select none

### Bootloader

d-i grub-installer/only_debian boolean true

### Configure the installed system

d-i preseed/late_command string \
    in-target usermod -aG sudo initial; \
    mkdir -p /target/home/initial/.ssh; \
    chmod 0700 /target/home/initial/.ssh; \
    printf '%s\n' 'YOUR_SSH_PUBLIC_KEY' > /target/home/initial/.ssh/authorized_keys; \
    in-target chown -R initial:initial /home/initial/.ssh; \
    in-target chmod 0600 /home/initial/.ssh/authorized_keys; \
    mkdir -p /target/etc/ssh/sshd_config.d; \
    in-target sh -c 'printf "%s\n" "PermitRootLogin no" "PubkeyAuthentication yes" "PasswordAuthentication yes" "KbdInteractiveAuthentication no" "UsePAM yes" > /etc/ssh/sshd_config.d/10-local-auth.conf'; \
    in-target sh -c 'printf "%s\n" "#!/usr/sbin/nft -f" "flush ruleset" "" "table inet filter {" "    chain input {" "        type filter hook input priority filter;" "        policy drop;" "        iifname lo accept" "        ct state invalid drop" "        ct state established,related accept" "        ip protocol icmp accept" "        meta l4proto ipv6-icmp accept" "        tcp dport 22 ct state new accept" "    }" "    chain forward {" "        type filter hook forward priority filter;" "        policy drop;" "    }" "    chain output {" "        type filter hook output priority filter;" "        policy accept;" "    }" "}" > /etc/nftables.conf'; \
    in-target systemctl enable ssh.service; \
    in-target systemctl enable nftables.service
```



## What happens during installation

1. **Boot Ventoy**

   Select the Debian netinst ISO and its preseed configuration.

2. **Debian Installer**

   Network configuration and package retrieval occur. Disk partitioning remains interactive.

3. **Base installation**

   Debian installs the base system, SSH server, sudo and nftables.

4. **Late command**

   The installer creates the SSH key directory, configures authentication, creates the firewall rules and enables the services.

5. **First boot**

   The machine starts with SSH and nftables enabled.

## Verify the installation

Log in locally or through SSH and run:

```
# Confirm sudo membership
id initial

# Confirm sudo works
sudo -v

# Validate SSH configuration
sudo sshd -t

# Check effective SSH settings
sudo sshd -T | grep -E \
  'permitrootlogin|passwordauthentication|pubkeyauthentication'

# Inspect firewall
sudo nft list ruleset

# Check services
systemctl status ssh
systemctl status nftables
```

From another machine, test:

```
ssh initial@SERVER_IP
```

Then test password authentication independently:

```
ssh -o PreferredAuthentications=password \
    -o PubkeyAuthentication=no initial@SERVER_IP
```



## important considerations

**Disk selection:** I have intentionally not automated partitioning. We should use a specific disk identifier such as `/dev/disk/by-id/...` when you are ready to automate that part. Never assume `/dev/sda` is the intended disk, particularly when installing from USB.

**Network:** Netinst needs working network connectivity and access to a Debian package mirror. The firewall is enabled late in the installation, after package retrieval.

**SSH password authentication:** This configuration enables both key and password authentication for `initial`. Root can authenticate locally but cannot log in over SSH.

**Testing:** Test this in an Incus VM first. In particular, verify the late-command execution and firewall rules before deploying to physical hardware.

The approach follows Debian's official Trixie preseed documentation and Ventoy's documented Debian preseed integration. 



## Suggested refinements

I would eventually move the SSH and firewall configuration into shared files in your Ansible repository, and have the preseed install only the minimum bootstrap configuration. Ansible can then own the ongoing system state rather than having two competing configuration authorities.

That gives you a clean division:

- **Debian Installer:** operating system, accounts, initial SSH access.
- **Ansible:** baseline configuration, security policy, updates and services.

It also makes it much easier to maintain the same baseline across Debian and Ubuntu.