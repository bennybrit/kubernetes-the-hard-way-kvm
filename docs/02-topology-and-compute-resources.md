# Topology Overview and Provisioning the Compute Resources
Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab, we'll provision the compute resources (VMs) required for running a highly available Kubernetes cluster on top of an RHEL based KVM host.

## Topology and Resources Overview
For our Kubernetes cluster, we'll use a total of 7 VMs, which will be created on top of the KVM host, the VMs will be split into 3 roles:

- **x1 k8s-lb** - Will be used for exposing the Kubernetes API to clients and load balancing the requests between the different controller nodes
- **x3 k8s-control** - Will be used for hosting the Kubernetes control plane components
- **x3 k8s-worker** - Worker machines (nodes), will be used for running containerized applications
- **x1 k8s-nfs** - NFS Server, will be used to provide storge space for the different Kubernetes storage resources

### Resources requirments
> You may adjust the resource allocations, but it's recommended to have at least 2 vCPUs and 2048 MB of RAM per node,
> also you may need to adjust the IP address allocation based on your network topology.

|   VM Name      |      Role     |  IP Address  | vCPUs | Memory (MB) |
| -------------- | ------------- | ------------ | ----- | ----------- |
| k8s-kvm-host   | KVM Host      | 172.19.33.1  |       |             |
| k8s-lb-01      | Load Balancer | 172.19.33.10 |   8   |    8192     |
| k8s-control-01 | Control Plane | 172.19.33.11 |   8   |    8192     |
| k8s-control-02 | Control Plane | 172.19.33.12 |   8   |    8192     |
| k8s-control-03 | Control Plane | 172.19.33.13 |   8   |    8192     |
| k8s-control-01 | Worker node   | 172.19.33.14 |   8   |    8192     |
| k8s-control-02 | Worker node   | 172.19.33.15 |   8   |    8192     |
| k8s-control-03 | Worker node   | 172.19.33.16 |   8   |    8192     |
| k8s-nfs-01     | NFS Server    | 172.19.33.20 |   8   |    8192     |

### Topology diagram
![topology diagram](images/k8s-cluster-diagram.png)

## Provisioning the VMs
> For the provisioning of the different VMs we'll be using the template `rhel-8.5-template.qcow2` we created in the previous lab (it should be located under /var/lib/libvirt/images path on your KVM host)

### VMs creation
Generate qcow2 image for each VM based on `rhel-8.5-template.qcow2` template:
```
{
  k8s_nodes="k8s-lb-01 k8s-control-01 k8s-control-02 k8s-control-03 k8s-worker-01 k8s-worker-02 k8s-worker-03 k8s-nfs-01"
  for node in ${k8s_nodes}; do
    cp /var/lib/libvirt/images/rhel-8.5-template.qcow2 /var/lib/libvirt/images/${node}.qcow2
  done
  sync
}
```
Create the different VMs using the `virt-install` tool (you may need to adjust the `--network` flag to fit your network topology):
```
{
  k8s_nodes="k8s-lb-01 k8s-control-01 k8s-control-02 k8s-control-03 k8s-worker-01 k8s-worker-02 k8s-worker-03 k8s-nfs-01"
  for node in ${k8s_nodes}; do
    virt-install --import \
    --name ${node} \
    --vcpus 8 \
    --memory 8192 \
    --disk path=/var/lib/libvirt/images/${node}.qcow2 \
    --os-variant rhel8.5 \
    --network bridge=br-mgmt \
    --noautoconsole
  done
}
```
Verify that all VMs created successfully and up and running:
```
virsh list
```
> Expected output:
```
 Id   Name             State
--------------------------------
 1    k8s-lb-01        running
 2    k8s-control-01   running
 3    k8s-control-02   running
 4    k8s-control-03   running
 5    k8s-worker-01    running
 6    k8s-worker-02    running
 7    k8s-worker-03    running
 8    k8s-nfs-01       running
 ```
You can access the VMs console using the `virt-manager` GUI friendly tool or attach directly to the console from the host using the `virsh console` command (to detach from the console using the ctrl+shift+5 key combination)

### Initial VM configuration
Connect to each VM using a console and set its hostname and IP address (for example setting the `k8s-control-01` VM):
```
# attach a console (you'll be asked for a login name and password)
virsh console k8s-control-01

# set hostname and network configuration
hostnamectl set-hostname k8s-control-01
nmcli c modify enp1s0 ipv4.addresses 172.19.33.11/16 ipv4.method manual
nmcli c modify enp1s0 ipv4.gateway 172.19.0.1
nmcli c modify enp1s0 ipv4.dns 172.19.0.1
nmcli c down enp1s0; nmcli c up enp1s0
```
At this point, you should be able to connect to the different VMs using an SSH client.

Connect to each VM and generate a new SSH key:
```
/usr/bin/ssh-keygen -v -t rsa -N "" -f /root/.ssh/id_rsa
```

Update the `/etc/hosts` file on all the VMs and also on the KVM host (you may need to adjust the IP addresses to match your network topology):
```
cat >> /etc/hosts <<EOF

172.19.33.10    k8s-lb-01
172.19.33.11    k8s-control-01
172.19.33.12    k8s-control-02
172.19.33.13    k8s-control-03
172.19.33.14    k8s-worker-01
172.19.33.15    k8s-worker-02
172.19.33.16    k8s-worker-03
172.19.33.20    k8s-nfs-01
EOF
```

Make sure swap is disabled on all the VMs (you may also need to remove the swap entry from /etc/fstab file):
```
swapoff -a
```

> **Important!** In case you're not using the VM template that we've prepared in the previous lab, make sure to install the required dependencies/tools:
```
yum -y install tmux bash-completion tcpdump telnet vim yum-utils iproute-tc net-tools ipvsadm util-linux wget git util-linux jq \
               conntrack socat libcgroup container-selinux fuse-overlayfs fuse3 fuse3-libs libslirp slirp4netns haproxy nfs-utils
```

## Configuring SSH Access
SSH will be used to access and configure the different VMs in the next labs. I will be using the KVM host (`k8s-kvm-host`) as the client for accessing the different VM (but you can use a different client and connect to the VMs directly).

To access the VMs without being required to enter a password for each login, use the `ssh-copy-id` tool to install the KVM host SSH key on all the Kubernetes cluster VMs (in the following example I'm running it for the `k8s-control-01` node - make sure to run it for all the VMs):
```
[root@k8s-kvm-host ~]# ssh-copy-id root@k8s-control-01
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/root/.ssh/id_rsa.pub"
The authenticity of host 'k8s-control-01 (172.19.33.11)' can't be established.
ECDSA key fingerprint is SHA256:svD+qx1Q036cwQJjRGj34t6dE1rK4yuqCNfnUIRVDJU.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
root@k8s-lb-01's password: 

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'root@k8s-control-01'"
and check to make sure that only the key(s) you wanted were added.
```

## Verification

Before moving to the next lab, verify the following:
- All VMs are up and running
- You can access all the VMs using SSH
- The different VMs can ping each other
- Each VM has a unique hostname
- swap is disabled on all VMs
- NTP is synchronized on all VMs

Next: [Installing Container Runtime](03-container-runtime.md)
