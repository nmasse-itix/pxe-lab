# PXE Server Setup

Install dnsmasq, activate it and open the firewall ports.

```sh
dnf install dnsmasq
systemctl enable dnsmasq
firewall-cmd --add-service dhcp --permanent
firewall-cmd --add-service proxy-dhcp --permanent
firewall-cmd --add-service tftp --permanent
firewall-cmd --reload
```

Prepare the files to server over TFTP.

```sh
dnf install syslinux
mkdir -p /var/lib/tftpboot/pxelinux.cfg
cp /usr/share/syslinux/pxelinux.0 /var/lib/tftpboot/
cp /usr/share/syslinux/{menu,vesamenu,ldlinux,libcom32,libutil,reboot}.c32 /var/lib/tftpboot/
curl -Lo /tmp/shim.rpm http://ftp.pasteur.fr/mirrors/CentOS/8-stream/BaseOS/x86_64/os/Packages/shim-x64-15-15.el8_2.x86_64.rpm
curl -Lo /tmp/grub2-efi.rpm http://ftp.pasteur.fr/mirrors/CentOS/8-stream/BaseOS/x86_64/os/Packages/grub2-efi-x64-2.02-99.el8.x86_64.rpm
for i in *.rpm; do rpm2cpio $i | cpio -dimv; done
cp boot/efi/EFI/centos/shimx64.efi /var/lib/tftpboot/
cp boot/efi/EFI/centos/grubx64.efi /var/lib/tftpboot/
cp boot/efi/EFI/BOOT/BOOTX64.EFI /var/lib/tftpboot/
```

Add the CentOS Stream 8 files.

```sh
mkdir -p /var/lib/tftpboot/centos-stream-8/
curl -Lo CentOS-Stream-8-x86_64-20210311-boot.iso http://ftp.pasteur.fr/mirrors/CentOS/8-stream/isos/x86_64/CentOS-Stream-8-x86_64-20210311-boot.iso
mount -t iso9660 -o loop,ro /tmp/CentOS-Stream-8-x86_64-20210311-boot.iso /mnt
cp /mnt/images/pxeboot/{vmlinuz,initrd.img} /var/lib/tftpboot/centos-stream-8/
umount /mnt
```

Create the file **/var/lib/tftpboot/grub.cfg** (UEFI clients).

```
set timeout=60
menuentry 'CentOS Stream 8' {
  linuxefi centos-stream-8/vmlinuz ip=dhcp inst.repo=http://ftp.pasteur.fr/mirrors/CentOS/8-stream/BaseOS/x86_64/os/
  initrdefi centos-stream-8/initrd.img
}
```

Create the file **/var/lib/tftpboot/pxelinux.cfg/default** (BIOS clients).

```
DEFAULT menu.c32
PROMPT 1
TIMEOUT 60

LABEL centos8
 MENU LABEL Install ^CentOS Stream 8
 KERNEL centos-stream-8/vmlinuz
 APPEND initrd=centos-stream-8/initrd.img ip=dhcp inst.repo=http://ftp.pasteur.fr/mirrors/CentOS/8-stream/BaseOS/x86_64/os/

LABEL rescue
 MENU LABEL ^Rescue
 KERNEL centos-stream-8/vmlinuz
 APPEND initrd=centos-stream-8/initrd.img rescue

LABEL reboot
 MENU DEFAULT
 MENU LABEL Reboot
 COM32 reboot.c32

LABEL local
  MENU LABEL ^Boot from local drive
  LOCALBOOT 0xffff
```

Fix file permissions.

```
restorecon -RF /var/lib/tftpboot/
chmod -R go+rX /var/lib/tftpboot/
```
