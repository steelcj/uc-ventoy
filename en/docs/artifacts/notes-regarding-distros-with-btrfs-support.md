# Notes Regarding distros with BTRFS support

Yes, though it's been there longer than you might think, and it depends which edition you mean.

**Fedora.** Btrfs has been the default on the desktop editions since Fedora 33 in 2020. Fedora made Btrfs replace ext4 as the default filesystem in Fedora 33 for Workstation, KDE and the other desktop variants, and Fedora Cloud followed later. Fedora Server is the exception: At present, the default is XFS. The cloud change notes that Fedora Server, IoT, and CoreOS are still specifically excluded. You can still pick Btrfs manually in the Server installer. The usual explanation for the split is that Red Hat's kernel / filesystem group does not have much expertise in btrfs and is quite focused on XFS development.

**Rocky 9.** No Btrfs. Rocky follows RHEL, and a technology preview of Btrfs in RHEL 6 and 7 ended with the filesystem being dropped from RHEL 8 and onward. The Rocky forums are blunt about it: btrfs is not a supported file system in Rocky Linux. The workarounds are ELRepo's `kernel-ml` or the CentOS kmod repositories, which take you off the supported kernel. Treat Rocky as XFS on LVM.

**If you want Btrfs on the Enterprise Linux side:**

- **AlmaLinux 10.1 and later:** support covers both kernel and userspace enablement, and it is now possible to install AlmaLinux OS with a Btrfs filesystem from the very beginning. That applies to the 10.x series, not AlmaLinux 9.
- **Oracle Linux:** its UEK kernel includes Btrfs support.

**What it means for your Ventoy plan.** A Fedora or Rocky node would need a kickstart template, and since partitioning stays interactive in your baseline, the filesystem choice isn't urgent. The bigger difference is the firewall: Fedora and Rocky ship `firewalld` enabled by default. The kickstart would need to disable it so the bootstrap `nftables` ruleset, and later Foomuuri, is the only firewall manager. Unlike `ufw` on Ubuntu, `firewalld` is active out of the box, so this matters more.

Sources:

- [Btrfs Coming to Fedora 33 (Fedora Magazine)](https://fedoramagazine.org/btrfs-coming-to-fedora-33)
- [QA:Testcase Server filesystem default (Fedora Project Wiki)](https://fedoraproject.org/wiki/QA:Testcase_Server_filesystem_default)
- [Make btrfs the default file system for Fedora Cloud (Fedora Project Wiki)](https://fedoraproject.org/wiki/Changes/FedoraCloudBtrfsByDefault)
- [Fedora users mailing list on XFS versus Btrfs](https://mid.mail-archive.com/users@lists.fedoraproject.org/msg165853.html)
- [Rocky 9.2: enable BTRFS support in kernel (Rocky Linux Forum)](https://forums.rockylinux.org/t/rocky-9-2-enable-btrfs-support-in-kernel/11523)
- [Btrfs support coming to AlmaLinux 10.1 (LWN)](https://lwn.net/Articles/1043029)
- [AlmaLinux 10.1 launches with Btrfs filesystem support (Neowin)](https://www.neowin.net/amp/almalinux-101-launches-with-btrfs-filesystem-support/)
- [Oracle Linux 9 is a RHEL rebuild with built-in Btrfs support (The Register)](https://www.theregister.com/2022/07/12/oracle_linux_9/)
