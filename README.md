# Configuring a PXE, TFTP, DHCPD, HTTP server to automate Red Hat Virtualization Hosts Deployment on UEFI booting

## Preparing the Installation Environment
* [Download RHVH from Red Hat Portal and upload it to your pxe server](https://access.redhat.com/products/red-hat-virtualization#getstarted)

* Install httpd, tftp-server, and dhcp-server, configure firewall and start services
```console
root@utility:~# yum install -y httpd tftp-server dhcpd
root@utility:~# firewall-cmd --add-service={http,tftp,dhcp} --permanent
root@utility:~# firewall-cmd --runtime-to-permanent
root@utility:~# systemctl enable --now httpd dhcpd tftp
```
* Avail RHVH ISO from a network source
```console
root@utility:~# mkdir -pv /mnt/rhvh
root@utility:~# mount -o loop,ro -t iso9660 /path/to/image.iso /mnt/rhvh
root@utility:~# cp -r /mnt/rhvh/ /var/www/html/rhvh
```

* Extract squashfs.img from ISO
```console
root@utility:~# cp /mnt/rhvh/Packages/redhat-virtualization-host-image-update* /tmp
root@utility:~# cd /tmp
root@utility:~# rpm2cpio redhat-virtualization-host-image-update* | cpio -idmv
./usr/share/redhat-virtualization-host/image
./usr/share/redhat-virtualization-host/image/redhat-virtualization-host-4.5.3-202212070734_8.6.squashfs.img
./usr/share/redhat-virtualization-host/image/redhat-virtualization-host-4.5.3-202212070734_8.6.squashfs.img.meta
2170306 blocks
```

* Configure dhcpd

```console
root@utility:~# cat << EOF > /etc/dhcpd/dhcpd.conf 
#
# DHCP Server Configuration file.
#   see /usr/share/doc/dhcp-server/dhcpd.conf.example
#   see dhcpd.conf(5) man page
#
option space pxelinux;
option pxelinux.magic code 208 = string;
option pxelinux.configfile code 209 = text;
option pxelinux.pathprefix code 210 = text;
option pxelinux.reboottime code 211 = unsigned integer 32;
option architecture-type code 93 = unsigned integer 16;

subnet 10.0.0.0 netmask 255.255.255.0 { # <--- Your dhcp subnet and subnet mask
        option routers 10.0.0.254;      # <--- Your Gateway
        range 10.0.0.180 10.0.0.200;    # <--- Your DHCP range
        option domain-name "home.ins";  # <--- Your domain name
        class "pxeclients" {
          match if substring (option vendor-class-identifier, 0, 9) = "PXEClient";
          next-server 10.0.0.187;       # <--- Your TFTP Server

          if option architecture-type = 00:07 {
            filename "BOOTX64.EFI";
          } else {
            filename "pxelinux/pxelinux.0";
          }
  }
}
# If you want static ip leases
host rhvh1 {
    # MAC address of client computer
    hardware ethernet 00:50:56:b8:32:21;
    # static IP address
    fixed-address 10.0.0.51;
    option host-name "rhvh1";
}
EOF
```
* Copy RHV boot images to your tftp server directory
```console
root@utility:~#  mkdir -pv /var/lib/tftpboot/images/rhvh4.4
root@utility:~# cp /mnt/rhvh/images/pxeboot/{vmlinuz,initrd.img} /var/lib/tftpboot/images/rhvh4.4
```
* Copy ldlinux.c32 and pxelinux.0 to your tftp server directory
1. Create /var/lib/tftpboot/pxelinux directory
2. Download syslinux-tftpboot-version-architecture.rpm package (replace version-architecture with the one from your repository)
3. Extract pxelinux.0 and ldlinux.c32 files
```console
root@utility:~# mkdir -pv /var/lib/tftpboot/pxelinux
root@utility:~# cd /tmp
root@utility:~# yum search syslinux-tftpboot
======= Name Exactly Matched: syslinux-tftpboot =======
syslinux-tftpboot.noarch : SYSLINUX modules in /tftpboot, available for network booting
[root@utility tmp]# yum install --downloadonly --destdir . syslinux-tftpboot.noarch -y
[root@utility tmp]# ll
total 1085796
-r--r--r--. 1 root root 1111377302 Jan  5 03:00 redhat-virtualization-host-image-update-4.5.3-202212070734_8.6.x86_64.rpm
-rw-r--r--. 1 root root     473356 Jan  5 03:23 syslinux-tftpboot-6.04-5.el8.noarch.rpm  # <--- This is the one we want

[root@utility tmp]# rpm2cpio syslinux-tftpboot-6.04-5.el8.noarch.rpm | cpio -dimv
[root@utility tmp]# cp /tmp/tftpboot/{pxelinux.0,ldlinux.c32} /var/lib/tftpboot/pxelinux/
```
* Create grub.cfg file
```console
# cat << EOF > /var/tftpboot/grub.cfg
rd.net.timeout.carrier=60
set default=10
set timeout=3
menuentry  'Install RHVH 4.4' --class fedora --class gnu-linux --class gnu --class os {
   linuxefi images/rhvh4.4/vmlinuz inst.ks=http://<pxe-server-ip>/rhvh/ks.cfg inst.stage2=http://<pxe-server-ip>/rhvh quiet
   initrdefi images/rhvh4.4/initrd.img
}
EOF
```
