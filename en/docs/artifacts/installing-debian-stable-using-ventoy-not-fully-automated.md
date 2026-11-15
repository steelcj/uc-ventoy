# Installing Debian Stable Using Ventoy

## Description

This guide describes how to download, verify, and install Debian Stable using a Ventoy USB device.

The installation uses the official Debian installation ISO rather than a network installation image. The ISO is stored directly on the Ventoy data partition and selected from the Ventoy boot menu.

The procedure includes:

- Downloading the latest stable Debian installation image.
- Verifying the authenticity of Debian's checksum files.
- Verifying the ISO using SHA-512.
- Copying the ISO to a Ventoy USB device.
- Booting the installer using Ventoy.
- Installing a minimal Debian server without a desktop environment.

## 1. Installation requirements

| Requirement         | Description                                           |
| ------------------- | ----------------------------------------------------- |
| Target architecture | amd64 (64-bit Intel/AMD)                              |
| Distribution        | Debian Stable                                         |
| Release             | Debian 13.7.0 Trixie                                  |
| Installation media  | CD-1 ISO                                              |
| ISO size            | Approximately 700 MB                                  |
| USB device          | Ventoy-enabled USB                                    |
| Network             | Optional during installation; recommended for updates |

The CD-1 image contains the packages necessary for a standard Debian installation without requiring an Internet connection.

For a larger selection of packages during offline installation, the DVD-1 image is an alternative.

## 2. Download Debian Stable

### 2.1 Obtain the installation image

Visit the official Debian BitTorrent download page:

https://www.debian.org/CD/torrent-cd/

Select:

- Release: Stable
- Architecture: amd64
- Media: CD/USB
- Download method: BitTorrent

Download the torrent for the first CD image.

The expected ISO filename is:

```
debian-13.7.0-amd64-1.iso
```

Use a BitTorrent client such as Transmission or qBittorrent to download the ISO.

Do not use the `netinst` image. It is intended for network-based installation.

### 2.2 Download the verification files

From the same official Debian image directory, download:

- `SHA512SUMS`
- `SHA512SUMS.sign`

These files provide the checksum list and its OpenPGP signature.

Keep all three files together:

```
debian-downloads/
├── debian-13.7.0-amd64-1.iso
├── SHA512SUMS
└── SHA512SUMS.sign
```

Official verification documentation:

https://www.debian.org/CD/verify

## 3. Verify the downloaded files

There are two separate verification steps:

1. Verify the authenticity of the checksum file using its OpenPGP signature.
2. Verify the ISO contents using the SHA-512 checksum.

Both steps are important. A checksum downloaded without signature verification does not independently establish that the checksum came from Debian.

### 3.1 Verify the OpenPGP signature

On a Debian or Ubuntu system, install GnuPG if necessary:

```
sudo apt install gnupg
```

Obtain the Debian CD signing public key from the official verification page:

https://www.debian.org/CD/verify

Import the public key:

```
gpg --import debian-cd-signing-key.asc
```

Check its fingerprint against the fingerprint published on Debian's official verification page.

Then verify the signature:

```
gpg --verify SHA512SUMS.sign SHA512SUMS
```

A successful verification should report a good signature.

Do not accept a signature from an unverified or unexpected key.

### 3.2 Verify the ISO checksum

Change into the download directory:

```
cd debian-downloads
```

Run:

```
sha512sum -c --ignore-missing SHA512SUMS
```

Expected output:

```
debian-13.7.0-amd64-1.iso: OK
```

The `OK` result confirms that the downloaded ISO matches the SHA-512 value in the verified checksum file.

If verification fails, do not use the ISO. Download it again and repeat the verification procedure.

## 4. Prepare the Ventoy USB device

This guide assumes Ventoy is already installed on the USB device.

Ventoy allows ISO files to be copied directly onto its data partition. It does not require the ISO to be extracted or written to the USB device using a separate imaging utility.

### 4.1 Identify the USB device

Connect the Ventoy USB device.

Identify its partitions:

```
lsblk -o NAME,SIZE,FSTYPE,LABEL,MOUNTPOINTS
```

Locate the Ventoy data partition, normally the large exFAT partition.

Be careful not to confuse the USB device with the target installation disk.

### 4.2 Copy the ISO

Copy the verified ISO to the Ventoy data partition.

For example:

```
cp debian-13.7.0-amd64-1.iso /media/$USER/Ventoy/
```

The mount path may differ depending on the desktop environment and USB configuration.

Alternatively, use the file manager to copy the ISO.

The resulting USB directory might look like:

```
Ventoy/
└── debian-13.7.0-amd64-1.iso
```

There is no need to extract the ISO.

Safely eject the USB device when copying is complete.

## 5. Boot the Debian installer

### 5.1 Boot from USB

Insert the Ventoy USB device into the target computer.

Restart the computer and open its firmware boot menu.

Select the USB device.

The exact boot-menu key depends on the computer manufacturer. Common keys include:

- F12
- F11
- F10
- Esc

### 5.2 Select the Debian ISO

When Ventoy starts, it displays a list of available ISO files.

Select:

```
debian-13.7.0-amd64-1.iso
```

Choose **Normal Mode** when prompted.

Ventoy will boot the Debian installer directly from the ISO.

### 5.3 Start the installer

At the Debian boot menu, select:

```
Install
```

Alternatively, select:

```
Graphical install
```

Both options use the Debian Installer. The text-based installer is sufficient for a server installation.

## 6. Install Debian

Follow the installation prompts.

### 6.1 Language and location

Select:

- Language: English
- Location: Canada
- Locale: en_CA.UTF-8

Choose the appropriate keyboard layout.

### 6.2 Network configuration

If a network connection is available, configure it during installation.

For a server, use a reliable wired connection where possible.

If the installation is offline, continue without network access. When asked about using a network mirror, select the option to skip mirror configuration.

The base system can be installed from CD-1 without Internet access.

### 6.3 Hostname and domain

Enter the intended server hostname.

For example:

```
hostname: debian-server-001
```

Enter a domain name if applicable. Otherwise, leave it blank.

### 6.4 User accounts

Configure the root account and administrative user.

For a conventional sudo-based installation, leaving the root password blank disables direct root login and allows the first regular user to use sudo.

Use a strong password for the administrative user.

### 6.5 Disk partitioning

Select the intended installation disk carefully.

For a straightforward installation:

1. Select Guided partitioning.
2. Select the intended target disk.
3. Choose a partitioning scheme.
4. Review the proposed partition layout.
5. Confirm the changes only after checking the target disk.

For a basic server, a single partition with a separate swap area is a reasonable starting point.

**Warning:** Confirming the partitioning changes can erase existing data on the selected disk.

Do not select the Ventoy USB device as the installation target.

### 6.6 Package manager configuration

If a network connection is available, select a Debian mirror.

Otherwise, continue without configuring a network mirror.

The installation can be completed offline using the packages available on the installation media.

### 6.7 Software selection

For a minimal server installation, use the following selections:

| Software                   | Selection |
| -------------------------- | --------- |
| Debian desktop environment | Disabled  |
| GNOME                      | Disabled  |
| Xfce                       | Disabled  |
| KDE Plasma                 | Disabled  |
| SSH server                 | Enabled   |
| Standard system utilities  | Enabled   |

Avoid installing a desktop environment on a dedicated server unless there is a specific requirement.

If SSH server packages are unavailable from the installation media, install them after networking is configured:

```
sudo apt update
sudo apt install openssh-server
```

## 7. Install the bootloader

When prompted, install GRUB to the target system's boot device.

For a UEFI installation, the installer should configure the EFI System Partition on the target disk.

For a legacy BIOS installation, install GRUB to the target disk, not the Ventoy USB device.

Complete the installation and remove the Ventoy USB device when instructed.

Restart the computer.

## 8. Initial system verification

After the first boot, log in using the administrative account.

### 8.1 Verify the Debian release

```
cat /etc/debian_version
cat /etc/os-release
```

### 8.2 Verify the kernel

```
uname -a
```

### 8.3 Verify storage

```
lsblk -f
```

### 8.4 Verify network interfaces

```
ip -br address
```

### 8.5 Verify SSH

If SSH was installed:

```
systemctl status ssh
```

### 8.6 Update the system

Once network connectivity is available:

```
sudo apt update
sudo apt full-upgrade
```

Review proposed package changes before confirming.

## 9. Installation completion

At this point, Debian Stable should be installed and booting from the target system's storage.

The resulting system has:

- Debian Stable.
- No desktop environment.
- Standard system utilities.
- SSH server, if installed.
- A package-managed base system.
- A verified installation image.

The server is ready for subsequent configuration, including administrative access, security hardening, and Ansible management.

## References

- [Debian Stable Releases](https://www.debian.org/releases/)
- [Debian CD/DVD/USB Downloads](https://www.debian.org/CD/)
- [Debian BitTorrent Downloads](https://www.debian.org/CD/torrent-cd/)
- [Debian Image Verification](https://www.debian.org/CD/verify)
- [Debian Installation Guide](https://www.debian.org/releases/stable/amd64/)
- [Debian Installation Information](https://www.debian.org/releases/stable/debian-installer/)
- [Ventoy Project](https://www.ventoy.net/)
- [Ventoy Documentation](https://www.ventoy.net/en/doc_start.html)