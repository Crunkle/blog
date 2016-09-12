---
layout: post
title: Installing OVH servers visually via VNC
---

There are many dedicated hosting providers which lack features whilst installing operating systems to your servers. I recently tried using distribution kernels with the OVH installation system and ran into a dozen problems. This is a quick guide on how I installed my dedicated servers remotely using a virtual machine and VNC.

To start with, as this guide is tailored towards OVH servers, you need to turn off all server monitoring to prevent the ping timeout system from creating tickets when your server is in rescue mode.

![example](https://cdn.jared.im/static/firefox_2016-09-12_01-01-15.png)

Boot your server into any rescue mode and prepare to reinstall your server; now is the time to backup your data. The way we are going to install the server is using a virtual machine; attaching your drives, forwarding traffic and enabling VNC for remote installation. We will be using QEMU to emulate the installation as it will allow us to boot the installation ISO from rescue mode. You can compile this yourself if you are unable to find the precompiled binaries on your rescue system.

```shell
qemu-system-x86_64 --version
```

With QEMU ready to go, you will need to download your ISO to the server. As some rescue systems are booted via network drives, you will need to be patient as the download may take longer than expected.

```shell
wget http://buildlogs.centos.org/rolling/7/isos/x86_64/CentOS-7-x86_64-Minimal.iso
```

It is now time to start your virtual machine. You may need to tweak the following flags to suit your needs, but the flags should work fine if you have two drives on your machine. This command starts a new virtual system, using the ISO as a virtual CD-ROM, and forwards all traffic from port 80 of the host machine (the rescue system) to the virtual machine. You can see that VNC has been enabled on port 5901 (read the man page to understand why it is port 5901 and not just 1). The primary thing to note here is that we have attached two drives (hda and hdb) to the machine and will be booting from the CD-ROM. You can redact the second disk if you are not installing a RAID system.

```shell
qemu-system-x86_64 -net nic -net user,hostfwd=tcp::80-:80 -m 1024M -enable-kvm -hda /dev/sda -hdb /dev/sdb -vnc :1 -cdrom ~/CentOS-7-x86_64-Minimal.iso -boot d
```

If all goes well, you should now see that QEMU is running and you will be able to connect via VNC. If you are unable to connect to your server or your client crashes, you may want to try another client. I used TightVNC as it was compatible with the version of QEMU on my system.

![example](https://cdn.jared.im/static/javaw_2016-09-12_01-32-07.png)

You should now be able to install your system using your keyboard and mouse! The reason we forwarded traffic from port 80 becomes apparent here. There are many network time protocols that require port 80 to be open in order to fetch your approximate location. You can change this port, or open more ports if required. If you do not include this port then your operating system may have trouble connecting to the internet so you will have to connect at a later stage.

![example](https://cdn.jared.im/static/javaw_2016-09-09_02-42-54.png)

Configure the disk partitioning, boot drive and security settings before installing the server. Do not worry about the network configuration at this stage as it will be replaced later. Once installed, you will need to restart the server and terminate the QEMU process from earlier. It is now time to restart QEMU using different flags to boot from the hard drive instead. If you try to boot up your installed system now, you will be unable to connect as the network scripts will be pointing towards QEMU. You can boot from the hard drive by changing the boot flag and enabling SSH access by forwarding traffic from the rescue mode server (port 222) to the SSH server run inside of QEMU.

```shell
qemu-system-x86_64 -net nic -net user,hostfwd=tcp::222-:22 -m 1024M -enable-kvm -hda /dev/sda -hdb /dev/sdb -vnc :1 -boot c
```

You should now be able to connect to SSH on your installed server using the port 222. If you do not change the port, you will be connecting to the rescue mode server instead of the installed server. The next thing to do is setup the network scripts so that you can connect to the server remotely without QEMU or rescue mode. Be sure to delete the previous script created on installation as it will not work outside of the emulator.

```shell
[root@example ~]$ cat /etc/sysconfig/network-scripts/ifcfg-eno1
DEVICE=eno1
BOOTPROTO=none
ONBOOT=yes
GATEWAY=123.124.125.254
NETMASK=255.255.255.0
IPADDR=123.124.125.126
IPV6INIT=yes
IPV6_AUTOCONF=no
IPV6ADDR=0000:0000:0000:0000::/64
PEERDNS=yes
DNS1=8.8.8.8
DNS2=8.8.4.4
```

Once you have configured your networking, you are ready to reboot your server from the original drives and enjoy a freshly installed system.
