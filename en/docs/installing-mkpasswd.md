# Installing mkpasswd

Installing **mkpasswd** depends on your Linux distribution because it is not a standalone package but is bundled within other utilities. On **Debian**, **Ubuntu**, and **Linux Mint**, you must install the **whois** package:

```
sudo apt install whois
```

On **RHEL**, **CentOS**, **Fedora**, **Rocky Linux**, and **AlmaLinux**, the utility is typically found in the **expect** package:

```
sudo yum install expect
# or on newer Fedora/RHEL versions
sudo dnf install whois
```

For **Arch Linux**, install **whois** via:

```
sudo pacman -S whois
```

On **Alpine Linux**, use:

```
sudo apk add mkpasswd
```

After installation, verify the command is available by running `mkpasswd --help` or `which mkpasswd`. 