# Generating Kubernetes Configuration Files for Authentication
In this lab, we'll generate [Kubernetes configuration files](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/), also known as kubeconfig, which enable Kubernetes clients to locate and authenticate to the Kubernetes API Servers.

We'll generate the different kubeconfig files on the KVM host (`k8s-kvm-host`), and from it, we'll distribute them to the different Kubernetes cluster VMs.

## Client Authentication Configs
In this section we'll generate kubeconfig files for the `kube-proxy`, `kube-controller-manager`, and `kube-scheduler` clients and the `admin` user.

### Kubernetes Public IP Address
Each kubeconfig requires a Kubernetes API server to connect to. To support high availability the IP address assigned to the load balancer (`k8s-lb-01`) fronting the Kubernetes API servers will be used.

Set a variable with the IP address of the load balancer VM (`k8s-lb-01`):
```
LOADBALANCER_IP='172.19.33.10'
```

### The kube-proxy Kubernetes configuration File
Generate a kubeconfig file for the `kube-proxy` service:
```
{
  kubectl config set-cluster kubernetes \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://${LOADBALANCER_IP}:6443 \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-credentials system:kube-proxy \
    --client-certificate=kube-proxy.crt \
    --client-key=kube-proxy.key \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes \
    --user=system:kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
}
```

> The following file will be created: `kube-proxy.kubeconfig`

### The kube-controller-manager Kubernetes configuration File
Generate a kubeconfig file for the `kube-controller-manager` service:
```
{
  kubectl config set-cluster kubernetes \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.crt \
    --client-key=kube-controller-manager.key \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
}
```

> The following file will be created: `kube-controller-manager.kubeconfig`

### The kube-scheduler Kubernetes configuration File
Generate a kubeconfig file for the `kube-scheduler` service:
```
{
  kubectl config set-cluster kubernetes \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-credentials system:kube-scheduler \
    --client-certificate=kube-scheduler.crt \
    --client-key=kube-scheduler.key \
    --embed-certs=true \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes \
    --user=system:kube-scheduler \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
}
```

> The following file will be created: `kube-scheduler.kubeconfig`

### The admin user configuration File
Generate a kubeconfig file for the `admin` user:
```
{
  kubectl config set-cluster kubernetes \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=admin.kubeconfig

  kubectl config set-credentials admin \
    --client-certificate=admin.crt \
    --client-key=admin.key \
    --embed-certs=true \
    --kubeconfig=admin.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes \
    --user=admin \
    --kubeconfig=admin.kubeconfig

  kubectl config use-context default --kubeconfig=admin.kubeconfig
}
```

> The following file will be created: `admin.kubeconfig`

## Distribute the Kubernetes Configuration Files
Copy the `kube-controller-manager`, `kube-scheduler`, `kube-proxy`, and `admin` kubeconfig files to each controller VM:
```
for node in k8s-control-01 k8s-control-02 k8s-control-03; do
  scp kube-controller-manager.kubeconfig kube-scheduler.kubeconfig kube-proxy.kubeconfig admin.kubeconfig ${node}:~/
done
```

Copy the `kube-proxy` kubeconfig file to each worker VM:
```
for node in k8s-worker-01 k8s-worker-02 k8s-worker-03; do
  scp kube-proxy.kubeconfig ${node}:~/
done
```

Next: [Generating the Data Encryption Config and Key](07-data-encryption-keys.md)
