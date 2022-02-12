# Deploying the CNI Plugin

In this lab, we'll deploy the Kubernetes [cluster networking plugin](https://kubernetes.io/docs/concepts/cluster-administration/networking/), we'll be using the [Weave Net CNI](https://github.com/weaveworks/weave) plugin.

The commands in this lab will affect the entire cluster and only need to be run once from one of the controller VMs, so run them only from the `k8s-control-01` controller VM.

## Install CNI Plugin
Deploy Weave Net CNI plugin:
```
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

>Expected output:
```
serviceaccount/weave-net created
clusterrole.rbac.authorization.k8s.io/weave-net created
clusterrolebinding.rbac.authorization.k8s.io/weave-net created
role.rbac.authorization.k8s.io/weave-net created
rolebinding.rbac.authorization.k8s.io/weave-net created
daemonset.apps/weave-net created
```
> By default the range of IP addresses used by Weave Net is `10.32.0.0/12`

## Verification
Verify that Weave Net CNI Daemonset and Pods were deployed successfully:
```
kubectl get all -l name=weave-net -n kube-system
```

> Expected output:
```
NAME                  READY   STATUS    RESTARTS      AGE
pod/weave-net-hs4sf   2/2     Running   1 (30s ago)   53s
pod/weave-net-njbf2   2/2     Running   1 (30s ago)   53s
pod/weave-net-p524v   2/2     Running   1 (30s ago)   53s
pod/weave-net-plhjl   2/2     Running   1 (30s ago)   53s
pod/weave-net-qb6z8   2/2     Running   1 (30s ago)   53s
pod/weave-net-z2kwq   2/2     Running   1 (30s ago)   53s

NAME                       DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/weave-net   6         6         6       6            6           <none>          60s
```

List the registered worker nodes:
```
kubectl get nodes
```

> Expected output:
```
NAME             STATUS   ROLES                  AGE   VERSION
k8s-control-01   Ready    control-plane,master   63m   v1.23.3
k8s-control-02   Ready    control-plane,master   63m   v1.23.3
k8s-control-03   Ready    control-plane,master   63m   v1.23.3
k8s-worker-01    Ready    <none>                 63m   v1.23.3
k8s-worker-02    Ready    <none>                 63m   v1.23.3
k8s-worker-03    Ready    <none>                 63m   v1.23.3
```
> All the nodes should be in `Ready` state

Next: [Deploying the DNS Cluster Add-on](15-dns-addon.md)
