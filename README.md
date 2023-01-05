# Configuring a PXE, TFTP, DHCPD, HTTP server to automate Red Hat Virtualization Hosts Deployment on UEFI booting

## Preparing the Installation Environment
* Download RHVH from Red Hat Portal and upload it to your pxe server
https://access.redhat.com/products/red-hat-virtualization#getstarted

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
root@utility:~# cat << EOF > /etc/dhcpd/dhcpd.conf >
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

