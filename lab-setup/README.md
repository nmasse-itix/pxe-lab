# PXE Lab Setup

Create a dedicated network for the PXE lab with DHCP disabled.

```sh
sudo virsh net-define /dev/fd/0 <<EOF
<network>
  <name>pxe-lab</name>
  <forward mode='nat'>
    <nat>
      <port start='1024' end='65535'/>
    </nat>
  </forward>
  <bridge name='virbr2' stp='on' delay='0'/>
  <ip address='192.168.23.1' netmask='255.255.255.0'>
  </ip>
</network>
EOF
sudo virsh net-start pxe-lab
sudo virsh net-autostart pxe-lab
```

Install the PXE Server.

```sh
sudo virt-install -n pxe-server --memory 2048 --vcpus=1 --os-variant=centos8 --accelerate -v --disk path=/var/lib/libvirt/images/pxe-server.qcow2,size=10 -l $PWD/CentOS-Stream-8-x86_64-20210311-boot.iso --initrd-inject=$PWD/centos-ks.cfg --extra-args "ks=file:/centos-ks.cfg" --network network=pxe-lab
```

[Configure the PXE Server](../pxe-setup/README.md)

Test the PXE install of a BIOS client.

```sh
sudo virt-install -n pxe-client-bios --memory 2048 --vcpus=1 --os-variant=centos8 --accelerate -v --disk path=/var/lib/libvirt/images/pxe-client-bios.qcow2,size=10 --pxe --network network=pxe-lab
```

Test the PXE install of a UEFI client.

```sh
sudo virt-install -n pxe-client-uefi --memory 2048 --vcpus=1 --os-variant=centos8 --accelerate -v --disk path=/var/lib/libvirt/images/pxe-client-uefi.qcow2,size=10 --pxe --network network=pxe-lab --boot uefi
```

Clean up.

```sh
sudo virsh destroy pxe-client-uefi
sudo virsh undefine --nvram pxe-client-uefi
sudo rm /var/lib/libvirt/images/pxe-client-uefi.qcow2

sudo virsh destroy pxe-client-bios
sudo virsh undefine pxe-client-bios
sudo rm /var/lib/libvirt/images/pxe-client-bios.qcow2
```
