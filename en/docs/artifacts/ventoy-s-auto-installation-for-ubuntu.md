# Ventoy's Auto Installation for Ubuntu



Yes. **Ubuntu can be automated through Ventoy in essentially the same way as Debian**, but it uses a different unattended-installation system.

The main difference is the installer:

|                           | Debian           | Ubuntu Server   |
| ------------------------- | ---------------- | --------------- |
| Installer                 | Debian Installer | Subiquity       |
| Automation                | Preseed          | Autoinstall     |
| Configuration             | Preseed file     | Cloud-init YAML |
| Ventoy plugin             | Auto Install     | Auto Install    |
| Post-install provisioning | Ansible          | Ansible         |

## **Ubuntu Server approach**

Ubuntu Server uses an `autoinstall` configuration file, usually named `user-data`. You place it on the Ventoy USB and configure Ventoy's Auto Install plugin to associate it with the Ubuntu ISO.

A simplified configuration would look like this:

```
#cloud-config
autoinstall:
  version: 1

  locale: en_CA.UTF-8
  keyboard:
    layout: us

  timezone: America/Toronto

  identity:
    hostname: ubuntu-server
    username: initial
    realname: Initial User
    password: "$6$REPLACE_WITH_PASSWORD_HASH"

  ssh:
    install-server: true
    allow-pw: true
    authorized-keys:
      - "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... initial@workstation"

  packages:
    - sudo
    - nftables

  user-data:
    disable_root: false

  late-commands:
    - curtin in-target --target=/target -- usermod -aG sudo initial
```

This is an outline, not yet a complete installation configuration. In particular, the SSH configuration and firewall rules still need to be installed and enabled.

## **Important differences**

1. **Root account:** Ubuntu normally locks the root account and uses `sudo`. We can explicitly configure root access if you want a local root account, while keeping SSH root login disabled.
2. **SSH:** Ubuntu supports both public-key and password authentication. The `allow-pw: true` setting enables password authentication.
3. **Firewall:** We can install and configure `nftables` just as with Debian. I would use it directly rather than introduce UFW if you want a consistent firewall implementation.
4. **Disk partitioning:** Ubuntu autoinstall supports fully automated storage configuration. I would still identify the target disk explicitly rather than assume `/dev/sda`.
5. **Minimal installation:** Ubuntu Server is a reasonable minimal starting point; avoid installing optional server features you don't need.

## **My recommendation**

For your intended workflow, I would use the same architecture for both distributions:

- Ventoy handles booting and selecting the installation configuration.
- The native installer performs the unattended OS installation.
- The installer creates `initial`, installs the SSH key, enables sudo, and establishes the initial firewall.
- Ansible takes over for baseline configuration and subsequent service provisioning.

That keeps the installation process distribution-specific while keeping the server configuration largely consistent.

One consideration: Ubuntu's autoinstall mechanism is more closely integrated with cloud-init and Curtin, whereas Debian's preseed is based on the older Debian Installer. Both are viable; neither requires rebuilding the ISO. 

www.ventoy.net+1



I can put together matching Debian and Ubuntu installation templates so that both produce the same minimal server baseline.