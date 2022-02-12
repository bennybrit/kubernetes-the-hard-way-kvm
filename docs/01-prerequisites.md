# Prerequisites and KVM host preparations

This set of labs will run through the process of installing a Kubernetes cluster on top of a virtualized RHEL/CentOS 8 based platform (both for the KVM host and the cluster VMs).

Some Linux knowledge is required (KVM host configuration, networking, SSL Certificates, etc.), as I'm not going to deep dive into how to install and configure the host or how to create the virtual machine image template (but I will do provide the steps in this lab), knowing the Kubernetes fundamentals is also required.

I will be using RHEL (Red Hat Enterprise Linux) 8 as the operating system for the KVM host and VMs, but you can choose any other distribution (CentOS, Ubuntu, etc.), you may need to adjust some of the commands based on your distro (for example apt instead of yum in case using ubuntu)

This lab can be installed on any hardware you choose, even on your laptop using VirtualBox or any other virtualization software (in such case you can skip the KVM Host preparation step), this kind of setup should be enough to provide a playground for learning and testing purposes.

As this whole guide is based on the RHEL platform, for the VMs I was using a custom-built RHEL image (I've provided the guidelines for creating the image below), but you can use RHEL pre-built QCOW2 image in case you want to skip this step.

## RHEL 8 KVM Host
### Install and prepare the KVM host

Install the host with RHEL 8.5 (Minimal install)

Register the host and attach a subscription to it:
```
subscription-manager register
subscription-manager attach --pool=<pool id>
subscription-manager release --set=8.5
```

Update the host packages with available updates:
```
yum -y update
```

install required dependencies/tools (add more if required):
```
yum -y install bash-completion tmux vim git jq nfs-utils
```

Install `libvirt` and some additional virtualization tools:
```
yum -y install virt-install virt-viewer virt-manager libguestfs-tools-c OVMF
```

Install x11 server (in case you are going to use the `virt-manager` utility):
```
yum -y install xorg-x11-server-Xorg xorg-x11-xauth xorg-x11-apps google-noto-sans-fonts.noarch
```

Stop and disable firewalld service (or configure firewalld/nftables as required):
```
systemctl stop firewalld
systemctl disable firewalld
```

Disable SELinux:
```
sed -i -e 's/SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
```

In case PCI Passthrough is required (SR-IOV), add `intel_iommu=on` to the kernel boot parameters:
```
grubby --args="intel_iommu=on" --update-kernel ALL
```

Reboot the host to apply changes

Reconnect to the host, enable and start the libvirtd service:
```
systemctl enable libvirtd.service
systemctl start libvirtd.service
```

To validate that the host is configured in a suitable way to run VMs, use the `virt-host-validate` utility:
```
virt-host-validate
```
> Expected output:
```
  QEMU: Checking for hardware virtualization                                 : PASS
  QEMU: Checking if device /dev/kvm exists                                   : PASS
  QEMU: Checking if device /dev/kvm is accessible                            : PASS
  QEMU: Checking if device /dev/vhost-net exists                             : PASS
  QEMU: Checking if device /dev/net/tun exists                               : PASS
  QEMU: Checking for cgroup 'cpu' controller support                         : PASS
  QEMU: Checking for cgroup 'cpuacct' controller support                     : PASS
  QEMU: Checking for cgroup 'cpuset' controller support                      : PASS
  QEMU: Checking for cgroup 'memory' controller support                      : PASS
  QEMU: Checking for cgroup 'devices' controller support                     : PASS
  QEMU: Checking for cgroup 'blkio' controller support                       : PASS
  QEMU: Checking for device assignment IOMMU support                         : PASS
  QEMU: Checking if IOMMU is enabled by kernel                               : PASS
```

### Configure the KVM host networking
Set hostname:
```
hostnamectl set-hostname k8s-kvm-host
```

The default `virbr0` bridge that is created with the installation of `libvirt` should be enough for this lab (it will also provide the VMs with DHCP assignments).

In case you want to use your external network (like I do in this lab), make sure to create an appropriate bridge and attach the VMs to it, for example:
```
nmcli c add type bridge autoconnect yes con-name br-mgmt ifname br-mgmt
nmcli c modify br-mgmt ipv4.addresses 172.19.5.10/16 ipv4.method manual
nmcli c modify br-mgmt ipv4.gateway 172.19.0.1
nmcli c modify br-mgmt ipv4.dns 172.19.0.1
nmcli c delete eno1; nmcli c add type bridge-slave autoconnect yes con-name eno1 ifname eno1 master br-mgmt; nmcli c up eno1; nmcli c up br-mgmt
```

## RHEL 8 VM template
On the KVM host create a qcow2 image (in this example I've created it with the size of 150 GB):
```
qemu-img create -f qcow2 /var/lib/libvirt/images/rhel-8.5-template.qcow2 150G
```

Configure a new VM (name it "rhel-8.5-template") and use the "rhel-8.5-template.qcow2" image created in the previous step as the Disk

Install the VM with RHEL 8.5 (Minimal install), when setting the disk partitioning make sure to disable the swap partition.

Register the VM and attach a subscription to it:
```
subscription-manager register
subscription-manager attach --pool=<pool id>
subscription-manager release --set=8.5
```

Update the VM packages with available updates:
```
yum -y update
```

Install required dependencies/tools:
```
yum -y install tmux bash-completion tcpdump telnet vim yum-utils iproute-tc net-tools ipvsadm util-linux wget git util-linux jq \
               conntrack socat libcgroup container-selinux fuse-overlayfs fuse3 fuse3-libs libslirp slirp4netns haproxy nfs-utils
```

Stop and disable firewalld service:
```
systemctl stop firewalld
systemctl disable firewalld
```

Disable SELinux:
```
sed -i -e 's/SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
```

Enable serial console access:
```
grubby --args "console=tty0 console=ttyS0,115200n8" --update-kernel ALL
```

Shutdown the VM

From the KVM host run "virt-sysprep" to reset the VM so that clones can be made from it:
```
virt-sysprep -d rhel-8.5-template
```
> The `rhel-8.5-template.qcow2` image will be used in the next lab for creating the different Kubernetes cluster VMs.

Next: [Topology Overview and Provisioning the Compute Resources](02-topology-and-compute-resources.md)
