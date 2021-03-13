# PXE Server Setup

Install dnsmasq, activate it and open the firewall ports.

```sh
dnf install dnsmasq
systemctl enable dnsmasq
systemctl start dnsmasq
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

Add the Memtest files.

```sh
curl -Lo /tmp/memtest.gz http://www.memtest.org/download/5.31b/memtest86+-5.31b.bin.gz
gunzip /tmp/memtest.gz
mkdir -p /var/lib/tftpboot/memtest/
cp /tmp/memtest /var/lib/tftpboot/memtest/
```

Create the file **/var/lib/tftpboot/grub.cfg** (UEFI clients).

```
set timeout=60

menuentry 'CentOS Stream 8' {
  linuxefi centos-stream-8/vmlinuz ip=dhcp inst.repo=http://ftp.pasteur.fr/mirrors/CentOS/8-stream/BaseOS/x86_64/os/
  initrdefi centos-stream-8/initrd.img
}

menuentry 'Rescue' {
  linuxefi centos-stream-8/vmlinuz rescue
  initrdefi centos-stream-8/initrd.img
}

menuentry 'Reboot' {
  reboot
}
```

Create the file **/var/lib/tftpboot/pxelinux.cfg/default** (BIOS clients).

```
DEFAULT menu.c32
PROMPT 0
TIMEOUT 600

LABEL centos8
 MENU LABEL Install ^CentOS Stream 8
 KERNEL centos-stream-8/vmlinuz
 APPEND initrd=centos-stream-8/initrd.img ip=dhcp inst.repo=http://ftp.pasteur.fr/mirrors/CentOS/8-stream/BaseOS/x86_64/os/

LABEL rescue
 MENU LABEL ^Rescue
 KERNEL centos-stream-8/vmlinuz
 APPEND initrd=centos-stream-8/initrd.img rescue

LABEL Memtest
 MENU LABEL Memtest
 KERNEL memtest/memtest

LABEL reboot
 MENU DEFAULT
 MENU LABEL Reboot
 COM32 reboot.c32

LABEL local
  MENU LABEL ^Boot from local drive
  LOCALBOOT 0xffff
```

Fix file permissions.

```sh
restorecon -RF /var/lib/tftpboot/
chmod -R go+rX /var/lib/tftpboot/
```

## Automated install based on Mac Address

Create **/var/lib/tftpboot/pxelinux.cfg/01-52-54-00-88-a4-b0**.

```sh
DEFAULT menu.c32
PROMPT 0
TIMEOUT 50

LABEL centos8
 MENU DEFAULT
 MENU LABEL Install CentOS Stream 8 with Kickstart
 KERNEL centos-stream-8/vmlinuz
 APPEND initrd=centos-stream-8/initrd.img ip=dhcp inst.repo=http://ftp.pasteur.fr/mirrors/CentOS/8-stream/BaseOS/x86_64/os/ inst.ks=http://192.168.23.10/auto-ks.cfg
```

Create **/var/lib/tftpboot/grub.cfg-01-52-54-00-88-a4-b0**.

```sh
set timeout=5

menuentry 'Install CentOS Stream 8 with Kickstart' {
  linuxefi centos-stream-8/vmlinuz ip=dhcp inst.repo=http://ftp.pasteur.fr/mirrors/CentOS/8-stream/BaseOS/x86_64/os/ inst.ks=http://192.168.23.10/auto-ks.cfg
  initrdefi centos-stream-8/initrd.img
}
```

Install lighttpd.

```sh
dnf -y install epel-release
systemctl enable lighttpd
systemctl start lighttpd
firewall-cmd --add-service http --permanent
firewall-cmd --reload
```

Create **/var/www/lighttpd/auto-ks.cfg** from [auto-ks.cfg](auto-ks.cfg).

## DNS Setup

Update **/etc/dnsmasq.conf**.

```
# DNS
auth-zone=itix.lab,192.168.23.0/24
auth-server=ns.itix.lab,
auth-ttl=60
expand-hosts
no-hosts
addn-hosts=/etc/dnsmasq.hosts
domain=itix.lab
```

Create **/etc/ethers** and **/etc/dnsmasq.hosts**.

```sh
touch /etc/ethers /etc/dnsmasq.hosts
```

## French Keyboard Layout

### For Grub

Generate a suitable Keyboard layout for Grub.

```sh
mkdir -p /var/lib/tftpboot/EFI/centos/layouts
gunzip -c /usr/lib/kbd/keymaps/xkb/fr.map.gz |grub2-mklayout -o /var/lib/tftpboot/EFI/centos/layouts/fr.gkb
```

Edit **/var/lib/tftpboot/grub.cfg** and add the following lines.

```
insmod keylayouts
insmod at_keyboard
terminal_input at_keyboard
keymap fr
```

### For Syslinux

```sh
curl -Lo syslinux.tgz https://mirrors.edge.kernel.org/pub/linux/utils/boot/syslinux/syslinux-6.03.tar.gz
tar zxvf syslinux.tgz
cd syslinux-6.03
cat > patch <<"EOF"
diff --git a/utils/keytab-lilo b/utils/keytab-lilo
index 9e34160..337a869 100755
--- a/utils/keytab-lilo
+++ b/utils/keytab-lilo
@@ -48,9 +48,9 @@ sub load_map
     $empty = 1;
     while (<FILE>) {
 	chop;
-	if (/^(static\s+)?u_short\s+(\S+)_map\[\S*\]\s+=\s+{\s*$/) {
+	if (/^(static\s+)?(u_|unsigned )short\s+(\S+)_map\[\S*\]\s+=\s+{\s*$/) {
 	    die "active at beginning of map" if defined $current;
-	    $current = $pfx.":".$2;
+	    $current = $pfx.":".$3;
 	    next;
 	}
 	undef $current if /^};\s*$/;
EOF
patch -p1 < patch
./utils/keytab-lilo -p 60=46 -p 92=60 -p 124=62 /usr/lib/kbd/keymaps/legacy/i386/qwerty/us.map.gz /usr/lib/kbd/keymaps/legacy/i386/azerty/fr-pc.map.gz > /var/lib/tftpboot/fr-pc.ktl
```

Edit **/var/lib/tftpboot/pxelinux.cfg/default** and add the following lines.

```
KBDMAP fr-pc.ktl
```

More details here:
- https://wiki.archlinux.fr/syslinux
- http://ix.io/n0p

