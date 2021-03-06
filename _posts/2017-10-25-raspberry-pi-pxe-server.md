---
layout: post
title: "Raspberry Pi PXE Server"
tags: [raspberry-pi,pxe,kali,tftp,nfs]
---

## Prerequirements
This setup was tested on *[Raspbian Stretch](https://www.raspberrypi.org/downloads/raspbian/)*.
In this example the Raspberry Pi's IP address is *192.168.1.10/24*.

First install the required packages:
```bash
apt-get update
apt-get install -y vim dnsmasq pxelinux
```

## dnsmasq
`dnsmasq` is used as a DHCP proxy here, it uses an existing DHCP server on the network for assigning IPs. Also, the DNS service is disabled.

1. Edit `/etc/dnsmasq.conf`:
   ```
   port=0
   interface=eth0
   dhcp-range=192.168.1.0,proxy,255.255.255.0
   dhcp-script=/bin/echo
   pxe-service=x86PC, "PXE Boot Menu", pxelinux
   dhcp-boot=pxelinux.0
   enable-tftp
   tftp-root=/var/lib/tftpboot
   ```
2. Create directories:
   ```bash
   mkdir -p /var/lib/tftpboot/pxelinux.cfg
   ```
3. Restart service:
   ```bash
   systemctl restart dnsmasq
   ```

## pxelinux (the bootloader)
1. Edit `/var/lib/tftpboot/pxelinux.cfg/default`:
   ```
   MENU TITLE Network Boot Menu
   DEFAULT menu.c32
   ```
2. Create file links:
   ```bash
   ln -s /usr/lib/PXELINUX/pxelinux.0 /var/lib/tftpboot/
   ln -s /usr/lib/syslinux/modules/bios/{ldlinux,menu,libcom32,libutil}.c32 /var/lib/tftpboot/
   ```

## PXE bootable Kali Linux 
1. Download *Kali Linux* ISO:
   ```bash
   wget http://cdimage.kali.org/kali-2017.2/kali-linux-2017.2-amd64.iso
   ```
2. Create mount directories:
   ```bash
   mkdir -p /var/lib/tftpboot/kali
   mkdir -p /mnt/kali
   ```
3. Mount ISO:
   ```bash
   mount kali-linux-2017.2-amd64.iso /mnt/kali
   ```
4. Copy files from ISO and unmount:
   ```bash
   cp /mnt/kali/live/{filesystem.squashfs,initrd.img,vmlinuz} /var/lib/tftpboot/kali/
   umount /mnt/kali
   ```
5. Append to `/var/lib/tftpboot/pxelinux.cfg/default`:
   ```

   LABEL kali
   MENU default
   MENU LABEL Kali Linux
   KERNEL /kali/vmlinuz
   APPEND initrd=/kali/initrd.img boot=live username=root hostname=kali gfxpayload=1024x768x16,1024x768 components fetch=tftp://192.168.1.10/kali/filesystem.squashfs
   ```

Now it's possible to boot *Kali Linux* over the network:
![raspberry-pi-pxe-server](/files/raspberry-pi-pxe-server.png)

## Update: Boot from NFS share
This configuration downloads the whole Kali live-system via TFTP, which is incredibly slow. To speed up the boot process, it can be accessed via NFS instead of the anachronistic TFTP protocol.
Follow those steps to boot from an NFS share:

1. Install packages:
   ```bash
   apt-get install -y nfs-kernel-server
   ```
2. Edit `/etc/exports`:
   ```
   /var/lib/tftpboot/kali 192.168.1.0/24(ro,sync,no_subtree_check,no_root_squash)
   ```
3. Reload `exports`:
   ```bash
   exportfs -ra
   ```
4. Enable & start NFS service:
   ```bash
   systemctl enable nfs-kernel-server
   systemctl start nfs-kernel-server
   ```
5. Edit *APPEND*-line in `/var/lib/tftpboot/pxelinux.cfg/default`:
   ```
   APPEND initrd=/kali/initrd.img boot=live username=root hostname=kali gfxpayload=1024x768x16,1024x768 components netboot=nfs nfsroot=192.168.1.10:/var/lib/tftpboot/kali live-media-path=/
   ```

---
1. [https://github.com/pimterry/rpi-pxe-server](https://github.com/pimterry/rpi-pxe-server)
2. [http://www.raspberry-pi-geek.de/Magazin/2014/02/Raspberry-Pi-als-PXE-Server/](http://www.raspberry-pi-geek.de/Magazin/2014/02/Raspberry-Pi-als-PXE-Server/)
3. [http://www.gtkdb.de/index_7_2744.html](http://www.gtkdb.de/index_7_2744.html)
4. [https://lists.debian.org/debian-live/2015/05/msg00028.html](https://lists.debian.org/debian-live/2015/05/msg00028.html)

