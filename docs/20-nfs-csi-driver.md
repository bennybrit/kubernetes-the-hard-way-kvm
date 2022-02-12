# Deploying the NFS CSI Driver

In this lab, we'll deploy an NFS [CSI Driver](https://kubernetes.io/blog/2019/01/15/container-storage-interface-ga/) which will add the support for dynamic provisioning of Persistent Volumes via Persistent Volume Claims by creating a new subdirectory on the NFS server.

We'll deploy the [NFS CSI Driver](https://github.com/kubernetes-csi/csi-driver-nfs) in high availability mode.

The commands in this lab will affect the entire cluster and only need to be run once from one of the controller VMs, so run them only from the `k8s-control-01` controller VM.

## The NFS CSI Driver
Deploy NFS CSI Driver (v3.1.0):
```
kubectl apply -f https://raw.githubusercontent.com/bennybrit/kubernetes-the-hard-way/master/deployments/csi-driver-nfs-3.1.0.yaml
```
>Expected output:
```
serviceaccount/csi-nfs-controller-sa created
clusterrole.rbac.authorization.k8s.io/nfs-external-provisioner-role created
clusterrolebinding.rbac.authorization.k8s.io/nfs-csi-provisioner-binding created
csidriver.storage.k8s.io/nfs.csi.k8s.io created
deployment.apps/csi-nfs-controller created
daemonset.apps/csi-nfs-node created
```

List the Controller pods created by the `csi-driver-nfs` deployment:
```
kubectl get pods -l app=csi-nfs-controller -n kube-system -o wide
```
> Expected output:
```
NAME                                  READY   STATUS    RESTARTS   AGE     IP             NODE             NOMINATED NODE   READINESS GATES
csi-nfs-controller-57bd48c448-vsqt8   3/3     Running   0          5m15s   172.19.33.11   k8s-control-01   <none>           <none>
csi-nfs-controller-57bd48c448-t8vz2   3/3     Running   0          5m15s   172.19.33.12   k8s-control-02   <none>           <none>
csi-nfs-controller-57bd48c448-8x728   3/3     Running   0          5m15s   172.19.33.13   k8s-control-03   <none>           <none>
```

List the Node pods created by the `csi-driver-nfs` deployment:
```
kubectl get pods -l app=csi-nfs-node -n kube-system -o wide
```
> Expected output:
```
NAME                 READY   STATUS    RESTARTS   AGE     IP             NODE             NOMINATED NODE   READINESS GATES
csi-nfs-node-s5dtn   3/3     Running   0          5m15s   172.19.33.11   k8s-control-01   <none>           <none>
csi-nfs-node-66wpm   3/3     Running   0          5m15s   172.19.33.12   k8s-control-02   <none>           <none>
csi-nfs-node-wdkw4   3/3     Running   0          5m15s   172.19.33.13   k8s-control-03   <none>           <none>
csi-nfs-node-tg5qx   3/3     Running   0          5m15s   172.19.33.14   k8s-worker-01    <none>           <none>
csi-nfs-node-tg4g9   3/3     Running   0          5m15s   172.19.33.15   k8s-worker-02    <none>           <none>
csi-nfs-node-lvgnj   3/3     Running   0          5m15s   172.19.33.16   k8s-worker-03    <none>           <none>
```

Define a new [Storage Class](https://kubernetes.io/docs/concepts/storage/storage-classes/) to support dynamic provisioning:
```
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-csi
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: nfs.csi.k8s.io
parameters:
  server: k8s-nfs-01
  share: /k8s-nfs-pool
reclaimPolicy: Delete
volumeBindingMode: Immediate
mountOptions:
  - hard
  - nfsvers=4.2
EOF
```

List the availabe Storage Classes:
```
kubectl get storageclasses
```
> Expected output:
```
NAME                PROVISIONER      RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-csi (default)   nfs.csi.k8s.io   Delete          Immediate           false                  60s
```

## Verification
Deploy a dynamically provisioned PVC (Persistent Volume Claim) which will be mounted to a Pod running nginx:
```
kubectl apply -f https://raw.githubusercontent.com/bennybrit/kubernetes-the-hard-way/master/deployments/pvc-e2e-test.yaml
```
> Expected output:
```
persistentvolumeclaim/nginx-pvc created
pod/nginx-pod created
```

List the resources that have been created:
```
kubectl get pod,pvc -l k8s-app=pvc-e2e
```
> Expected output:
```
NAME            READY   STATUS    RESTARTS   AGE
pod/nginx-pod   1/1     Running   0          10s

NAME                              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/nginx-pvc   Bound    pvc-09ba4890-9212-49bb-a982-9b5fb1386f84   10Gi       RWO            nfs-csi        10s
```

Verify the mount point from within the Pod:
```
kubectl exec nginx-pod -- bash -c "findmnt /var/www -o TARGET,SOURCE,FSTYPE"
```
> Expected output:
```
TARGET   SOURCE                                                            FSTYPE
/var/www k8s-nfs-01:/k8s-nfs-pool/pvc-09ba4890-9212-49bb-a982-9b5fb1386f84 nfs4
```

From the NFS Server (`k8s-nfs-01`), list the directories under the `/k8s-nfs-pool` directory and make sure that a subdirectory that matches the PVC was created:
```
ls -l /k8s-nfs-pool/
```
> Expected output:
```
drwxrwxrwx 2 root root 6 Feb 12 21:00 pvc-09ba4890-9212-49bb-a982-9b5fb1386f84
```

Delete the resources used for verification:
```
{
  kubectl delete pod nginx-pod
  kubectl delete pvc nginx-pvc
}
```

Next: [Smoke Test](21-smoke-test.md)
