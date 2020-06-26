# KVM Cheatsheet

## Install KVM

Make sure KVM support (kvm_intel or kvm_amd) is available in your system

```bash
lsmod | grep kvm
$ lsmod | grep kvm
kvm_amd               110592  0
ccp                    98304  1 kvm_amd
kvm                   761856  1 kvm_amd
irqbypass              16384  1 kvm
```

Install KVM and tools
```bash
# core tools 
yum -y install qemu-kvm libvirt virt-install

# GUI management tools
yum -y install virt-manager virt-viewer

# disk image management tool
yum -y install libguestfs-tools
```

## Create Virtual Machine

### Create Linux VM from web install

```bash
mkdir /vmdata

virt-install \
--name centos7 \
--ram 4096 \
--disk path=/vmdata/centos7.img,size=30 \
--vcpus 2 \
--os-type linux \
--os-variant rhel7 \
--network network=default \
--graphics none \
--console pty,target_type=serial \
--location 'http://mirror.aarnet.edu.au/pub/centos/7.8.2003/os/x86_64/' \
--extra-args 'console=ttyS0,115200n8 serial'
```

### Create template from existing VM
```bash
# 'cento7' is the name of the original VM
virt-clone --original centos7 --name TMP_CENTOS7 --file /vmdata/TMP_CENTOS7.img
```

### Create VM from template
```bash
virt-clone --original TMP_CENTOS7 --name file-server --file /vmdata/file-sever.img
```

### Create VM from ISO
```bash

# create boot disk
qemu-img create -f qcow2 /vmdata/centos7.qcow2 10G

# create VM
virt-install --name centos7 \
 --vcpus 2 --ram 1024 \
 --disk path=/vmdata/centos7.qcow2,format=qcow2\
 --network network=default\
 --location /vmdata/CentOS-7-x86_64-Minimal-1810.iso\
 --os-type linux --os-variant centos7.0\
 --extra-args 'console=ttyS0,115200n8 serial'
```

## Image Operations

### Clone qcow2 image to block device (V2P)
```bash
qemu-img convert templates/CentOS-7-x86_64-GenericCloud-2003.qcow2 -O raw /dev/HostVG/openstack
```


