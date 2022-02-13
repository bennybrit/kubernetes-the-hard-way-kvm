# Bootstrapping the Kubernetes Worker Nodes

In this lab, we'll bootstrap the Kubernetes worker nodes. The following components will be installed on each node: [kubelet](https://kubernetes.io/docs/admin/kubelet), and [kube-proxy](https://kubernetes.io/docs/concepts/cluster-administration/proxies).

The worker components will be installed on the controller VMs (`k8s-control-01`, `k8s-control-02`, `k8s-control-03`) and worker VMs (`k8s-worker-01`, `k8s-worker-02`, `k8s-worker-03`), so make sure you run the upcoming steps on each one.

The controller VMs will be used to run management/control Pods, so we'll isolate them by setting the required labels and taints to prevent the scheduler from scheduling standard workload Pods on them.

## Provisioning a Kubernetes Worker Node
Create the working directories:
```
mkdir -p /etc/cni/net.d \
         /opt/cni/bin \
         /var/lib/kubelet \
         /var/lib/kube-proxy \
         /var/lib/kubernetes \
         /var/run/kubernetes \
         /etc/kubernetes/pki
```

### Download and install the Kubernetes worker binaries
Download the official Kubernetes release binaries (v1.23.3) and set executable permissions:
```
{
  wget https://dl.k8s.io/v1.23.3/bin/linux/amd64/kubelet -O /usr/local/bin/kubelet
  wget https://dl.k8s.io/v1.23.3/bin/linux/amd64/kube-proxy -O /usr/local/bin/kube-proxy
  chmod +x /usr/local/bin/kubelet \
           /usr/local/bin/kube-proxy
}
```

### Download and install CNI Plugins
Download and install [CNI Plugins](https://github.com/containernetworking/plugins) (v1.0.1):
```
{
  wget https://github.com/containernetworking/plugins/releases/download/v1.0.1/cni-plugins-linux-amd64-v1.0.1.tgz
  tar -xvf cni-plugins-linux-amd64-v1.0.1.tgz -C /opt/cni/bin/
  rm -f cni-plugins-linux-amd64-v1.0.1.tgz
}
```

### Distribute TLS cerficates and configuration files
```
{
  \cp ca.crt /etc/kubernetes/pki/
  \cp kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
}
```

### Generate kubelet kubeconfig file
Set a variable with the IP address of the load balancer VM (`k8s-lb-01`):
```
LOADBALANCER_IP='172.19.33.10'
```

Generate a kubelet kubeconfig file which will be used to bootstrap the kubelet using token:
> Make sure that the token id matches the token id we've created in [Configuring the Kubernetes Control Plane lab](10-configuring-kubernetes-controllers.md)
```
{
  kubectl config set-cluster kubernetes \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://${LOADBALANCER_IP}:6443 \
    --kubeconfig=/var/lib/kubelet/bootstrap-kubelet.kubeconfig

  kubectl config set-credentials bootstrap-kubelet \
    --token=07401b.f395accd246ae52d \
    --kubeconfig=/var/lib/kubelet/bootstrap-kubelet.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes \
    --user=bootstrap-kubelet \
    --kubeconfig=/var/lib/kubelet/bootstrap-kubelet.kubeconfig

  kubectl config use-context default --kubeconfig=/var/lib/kubelet/bootstrap-kubelet.kubeconfig
}
```

### Configure the Kubelet
Create the `kubelet-config.yaml` configuration file:
```
cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
authorization:
  mode: Webhook
registerNode: true
rotateCertificates: true
serverTLSBootstrap: true
cgroupDriver: systemd
clusterDomain: cluster.local
clusterDNS:
  - 10.96.0.10
EOF
```

Create the `kubelet.service` systemd unit file:
```
cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=docker.service
Requires=docker.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --bootstrap-kubeconfig=/var/lib/kubelet/bootstrap-kubelet.kubeconfig \\
  --config=/var/lib/kubelet/kubelet-config.yaml \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --network-plugin=cni \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Configure the Kubernetes Proxy
Create the `kube-proxy-config.yaml` configuration file:
```
cat <<EOF | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: /var/lib/kube-proxy/kubeconfig
mode: iptables
clusterCIDR: 10.244.0.0/16
EOF
```

Create the `kube-proxy.service` systemd unit file:
```
cat <<EOF | sudo tee /etc/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --config=/var/lib/kube-proxy/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Start the Worker Services
```
{
  sudo systemctl daemon-reload
  sudo systemctl enable kubelet kube-proxy
  sudo systemctl start kubelet kube-proxy
}
```

> Remember to run the above commands on each worker VM: `k8s-worker-01`, `k8s-worker-02`, and `k8s-worker-03`.

## Approve CSRs
The commands in this section will affect the entire cluster and only need to be run once from one of the controller VMs, so run them only from the `k8s-control-01` controller VM.

List pending CSRs for the different controller and worker VMs:
```
kubectl get csr --kubeconfig admin.kubeconfig
```

> Expected output:
```
NAME        AGE     SIGNERNAME                                    REQUESTOR                    REQUESTEDDURATION   CONDITION
csr-85lrt   2m30s   kubernetes.io/kubelet-serving                 system:node:k8s-control-01   <none>              Pending
csr-h6t7q   2m30s   kubernetes.io/kubelet-serving                 system:node:k8s-control-02   <none>              Pending
csr-j2f24   2m30s   kubernetes.io/kubelet-serving                 system:node:k8s-control-03   <none>              Pending
csr-k75kz   2m30s   kubernetes.io/kubelet-serving                 system:node:k8s-worker-01    <none>              Pending
csr-tp45m   2m30s   kubernetes.io/kubelet-serving                 system:node:k8s-worker-02    <none>              Pending
csr-vh7qj   2m30s   kubernetes.io/kubelet-serving                 system:node:k8s-worker-03    <none>              Pending
```

Approve all the pending CSRs, for example:
```
kubectl --kubeconfig admin.kubeconfig certificate approve csr-85lrt csr-h6t7q csr-j2f24 csr-k75kz csr-tp45m csr-vh7qj
```

## Control plane nodes isolation
To prevent the scheduler from scheduling standard workload Pods on the controller nodes, we'll need to add a predefined set of taints and labels to them.

The commands in this section will affect the entire cluster and only need to be run once from one of the controller VMs, so run them only from the `k8s-control-01` controller VM.

Add the `node-role.kubernetes.io/master:NoSchedule` taint to the controller nodes:
```
{
  kubectl taint nodes k8s-control-01 node-role.kubernetes.io/master:NoSchedule --kubeconfig admin.kubeconfig
  kubectl taint nodes k8s-control-02 node-role.kubernetes.io/master:NoSchedule --kubeconfig admin.kubeconfig
  kubectl taint nodes k8s-control-03 node-role.kubernetes.io/master:NoSchedule --kubeconfig admin.kubeconfig
}
```

Add the `node-role.kubernetes.io/control-plane=` and `node-role.kubernetes.io/master=` labels to the controller nodes:
```
{
  kubectl label nodes k8s-control-01 node-role.kubernetes.io/control-plane= node-role.kubernetes.io/master= --kubeconfig admin.kubeconfig
  kubectl label nodes k8s-control-02 node-role.kubernetes.io/control-plane= node-role.kubernetes.io/master= --kubeconfig admin.kubeconfig
  kubectl label nodes k8s-control-03 node-role.kubernetes.io/control-plane= node-role.kubernetes.io/master= --kubeconfig admin.kubeconfig
}
```

## Verification
List the registered worker nodes:
```
kubectl get nodes --kubeconfig admin.kubeconfig
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

Next: [Configuring kubectl for Remote Access](13-configuring-kubectl.md)
