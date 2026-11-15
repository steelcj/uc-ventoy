---
title: "Btrfs Support by Distribution"
date: 2026-11-14
author: "Christopher Steel"
status: "Research note"
description: "Which distributions considered for uc-ventoy support Btrfs, and what that means for future kickstart targets."
---

# Btrfs Support by Distribution

## Fedora

Btrfs has been the default filesystem on Fedora's desktop editions, including Workstation and KDE, since Fedora 33 in 2020, replacing ext4 <a id="cite-fedora-magazine-2020"></a>([Fedora Magazine, 2020](#ref-fedora-magazine-2020)). Fedora Cloud followed later, while Fedora Server, IoT and CoreOS were explicitly excluded <a id="cite-fedora-cloud-btrfs"></a>([Fedora Project, n.d.-b](#ref-fedora-cloud-btrfs)). Fedora Server's default remains XFS <a id="cite-fedora-server-default"></a>([Fedora Project, n.d.-a](#ref-fedora-server-default)), although Btrfs can be chosen manually in its installer. The usual explanation for the split is that Red Hat's filesystem engineers focus on XFS <a id="cite-fedora-users-list"></a>([Fedora users mailing list, n.d.](#ref-fedora-users-list)).

## Rocky Linux 9

Rocky Linux follows RHEL, which dropped Btrfs from RHEL 8 onward after a technology preview in RHEL 6 and 7 <a id="cite-lwn-2025"></a>([LWN, 2025](#ref-lwn-2025)). Rocky's own forum confirms that Btrfs is not supported. The workarounds, ELRepo's `kernel-ml` or the CentOS kmod repositories, move a node off the supported kernel <a id="cite-rocky-forum"></a>([Rocky Linux Forum, n.d.](#ref-rocky-forum)). Treat Rocky as XFS on LVM.

## Btrfs on Enterprise Linux

AlmaLinux 10.1 and later support Btrfs in both the kernel and userspace, including installing onto Btrfs from the start. This applies to the 10.x series, not AlmaLinux 9 <a id="cite-neowin-2025"></a>([Neowin, 2025](#ref-neowin-2025)). Oracle Linux includes Btrfs support through its UEK kernel <a id="cite-register-2022"></a>([The Register, 2022](#ref-register-2022)).

## Relevance to uc-ventoy

A Fedora or Rocky target would need a kickstart template. Partitioning stays interactive in the raw node (RN-10), so the filesystem choice is not urgent. The larger difference is the firewall: Fedora and Rocky enable `firewalld` by default, so a kickstart template would have to disable it to meet RN-08. See the portability notes in [raw-node-baseline.md](../raw-node-baseline.md#portability-notes).

## References

<a id="ref-fedora-magazine-2020"></a>
Fedora Magazine. (2020). *Btrfs coming to Fedora 33*. https://fedoramagazine.org/btrfs-coming-to-fedora-33
[Return to citation](#cite-fedora-magazine-2020)

<a id="ref-fedora-server-default"></a>
Fedora Project. (n.d.-a). *QA:Testcase Server filesystem default*. Fedora Project Wiki. Retrieved November 14, 2026, from https://fedoraproject.org/wiki/QA:Testcase_Server_filesystem_default
[Return to citation](#cite-fedora-server-default)

<a id="ref-fedora-cloud-btrfs"></a>
Fedora Project. (n.d.-b). *Make btrfs the default file system for Fedora Cloud*. Fedora Project Wiki. Retrieved November 14, 2026, from https://fedoraproject.org/wiki/Changes/FedoraCloudBtrfsByDefault
[Return to citation](#cite-fedora-cloud-btrfs)

<a id="ref-fedora-users-list"></a>
Fedora users mailing list. (n.d.). *Re: filesystem choice for Fedora Server and Workstation* [Mailing list post]. Retrieved November 14, 2026, from https://mid.mail-archive.com/users@lists.fedoraproject.org/msg165853.html
[Return to citation](#cite-fedora-users-list)

<a id="ref-lwn-2025"></a>
LWN. (2025). *Btrfs support coming to AlmaLinux 10.1*. https://lwn.net/Articles/1043029
[Return to citation](#cite-lwn-2025)

<a id="ref-neowin-2025"></a>
Neowin. (2025). *AlmaLinux 10.1 launches with Btrfs filesystem support*. https://www.neowin.net/amp/almalinux-101-launches-with-btrfs-filesystem-support/
[Return to citation](#cite-neowin-2025)

<a id="ref-register-2022"></a>
The Register. (2022, July 12). *Oracle Linux 9 is a RHEL rebuild with built-in Btrfs support*. https://www.theregister.com/2022/07/12/oracle_linux_9/
[Return to citation](#cite-register-2022)

<a id="ref-rocky-forum"></a>
Rocky Linux Forum. (n.d.). *Rocky 9.2: Enable BTRFS support in kernel* [Online forum post]. Retrieved November 14, 2026, from https://forums.rockylinux.org/t/rocky-9-2-enable-btrfs-support-in-kernel/11523
[Return to citation](#cite-rocky-forum)
