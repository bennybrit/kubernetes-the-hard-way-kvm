# Cleaning Up

In this lab, we'll delete the compute resources created during this tutorial.

## Compute Instances
Connect to the KVM host (`k8s-kvm-host`)

Delete the load balancer, controllers, workers, and NFS server VMs:
```
{
  k8s_nodes="k8s-lb-01 k8s-control-01 k8s-control-02 k8s-control-03 k8s-worker-01 k8s-worker-02 k8s-worker-03 k8s-nfs-01"
  for node in ${k8s_nodes}; do
	  virsh destroy --domain ${node}
	  virsh undefine --domain ${node}
	  rm -f /var/lib/libvirt/images/${node}.qcow2
  done
}
```
