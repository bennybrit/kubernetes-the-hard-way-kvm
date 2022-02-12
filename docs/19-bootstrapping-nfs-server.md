# Bootstrapping the NFS Server

In this lab, we'll provision an NFS server which will be used to provide storage space for the different Kubernetes storage resources (Mounts, Volumes, Persistent Volumes, etc.)

The NFS server will be installed and configured on the NFS Server VM (`k8s-nfs-01`), so make sure you run the upcoming steps only on it.

## Install and configure NFS Server
Install the `nfs-utils` package:
```
yum -y install nfs-utils
```

Configure the NFS server to allow clients to use NFSv4 only:
```
cat > /etc/nfs.conf <<EOF
[nfsd]
tcp=y
vers2=n
vers3=n
vers4=y
vers4.0=y
vers4.1=y
vers4.2=y
EOF
```

Enable and start the `nfs-server` service:
```
{
  systemctl enable nfs-server.service
  systemctl start nfs-server.service
}
```

Create the directory that will be shared by the NFS server:
```
mkdir -p /k8s-nfs-pool
```

Allow the directory to be exported to remote client (our Kubernetes cluster VMs):
> My Kubernetes cluster VMs are using the `172.19.0.0/16` network subnet, so in this example, I'm allowing all clients from the `172.19.0.0/16` subnet to mount the share (you may need to adjust it to match your network topology)
```
cat > /etc/exports <<EOF
/k8s-nfs-pool 172.19.0.0/16(rw,no_root_squash,anonuid=1001,anongid=1001)
EOF
```

Export the new share specified in the `/etc/exports` file:
```
exportfs -av
```
> Expected output:
```
exporting 172.19.0.0/16:/k8s-nfs-pool
```

## Verification
Try to mount the share from one of the Kubernetes cluster VMs or the KVM host (`k8s-kvm-host`):
```
mount -o nfsvers=4 k8s-nfs-01:/k8s-nfs-pool /mnt
```

Verify that the share was mounted successfully:
```
df -h /mnt
```
> Expected output:
```
Filesystem                Size  Used Avail Use% Mounted on
k8s-nfs-01:/k8s-nfs-pool  150G  3.5G  147G   3% /mnt
```

Unmount the share:
```
umount /mnt
```

Next: [Deploying the NFS CSI Driver](20-nfs-csi-driver.md)
