# Configuring kubectl for Remote Access

In this lab, we'll generate a kubeconfig file for the `kubectl` command-line utility based on the `admin` user credentials.

We'll also configure the kubelet to use the load balancer as the frontend for Kubernetes API (instead of `localhost`), also we'll place the kubeconfig file under the `$HOME/.kube` directory to avoid providing the path to it each time we run the `kubectl` command.

As the `kubectl` utility was installed on the controller VMs (`k8s-control-01`, `k8s-control-02`, `k8s-control-03`) and worker VMs (`k8s-worker-01`, `k8s-worker-02`, `k8s-worker-03`), make sure you run the upcoming steps on each one of them.

In case you will be using the KVM host (`k8s-kvm-host`) also as your client, make sure to run the steps also on it.

## Generate the Admin Kubernetes Configuration File

Each kubeconfig requires a Kubernetes API Server to connect to. To support high availability the IP address assigned to the external load balancer fronting the Kubernetes API Servers will be used, set a variable with the IP address of the load balancer VM (`k8s-lb-01`):
```
LOADBALANCER_IP='172.19.33.10'
```

Generate a kubeconfig file suitable for authenticating as the `admin` user:

```
{
  kubectl config set-cluster kubernetes \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://${LOADBALANCER_IP}:6443 \

  kubectl config set-credentials admin \
    --client-certificate=admin.crt \
    --client-key=admin.key \
    --embed-certs=true \

  kubectl config set-context default \
    --cluster=kubernetes \
    --user=admin \

  kubectl config use-context default
}
```
> The following file will be created: `~/.kube/config`

## Cleanup
As we proceeded with the different labs, we've generated multiple files (configurations, certificates, keys, etc.) and distributed them to the different nodes, if you have followed the labs, those files should be located under the `~/` directory.

As we already bootstrapped the different controller and worker components, we can clean up those files, as this is also not recommended to keep them there due to security manners.

Remove all the differnt files by running the follwing on all the cluster nodes (load-balancer, controllers, workers, and KVM host):
```
{
  cd ~
  rm -f *.key *.crt *.csr *.cnf *.srl *.kubeconfig *.yaml
}
```

## Verification
List the nodes in the Kubernetes cluster (try to run it on each of the controller and worker nodes):
```
kubectl get nodes
```

> Expected output:
```
NAME             STATUS     ROLES                  AGE    VERSION
k8s-control-01   NotReady   control-plane,master   60s    v1.23.3
k8s-control-02   NotReady   control-plane,master   60s    v1.23.3
k8s-control-03   NotReady   control-plane,master   60s    v1.23.3
k8s-worker-01    NotReady   <none>                 60s    v1.23.3
k8s-worker-02    NotReady   <none>                 60s    v1.23.3
k8s-worker-03    NotReady   <none>                 60s    v1.23.3
```
> The nodes are expected to be in `NotReady` state, as we still haven't deployed any CNI plugin.

Next: [Deploying the CNI Plugin](14-cni-plugin.md)
