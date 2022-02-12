# Bootstrapping the Kubernetes Control Plane

In this lab, we'll bootstrap the Kubernetes control plane across three controller VMs and configure it for high availability. The following components will be installed on each controller VM: [Kubernetes API Server](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/), [Scheduler](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/), and [Controller Manager](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/).

The control plane components will be installed on the controller VMs (`k8s-control-01`, `k8s-control-02`, and `k8s-control-03`), so make sure you run the upcoming steps on each one.

## Provision the Kubernetes Control Plane
Create the Kubernetes configuration and work directories:
```
mkdir -p /etc/kubernetes/pki \
         /var/lib/kubernetes 
```

### Download and install the Kubernetes controller binaries
Download the official Kubernetes release binaries (v1.23.3) and set executable permissions:
```
{
  wget https://dl.k8s.io/v1.23.3/bin/linux/amd64/kube-apiserver -O /usr/local/bin/kube-apiserver
  wget https://dl.k8s.io/v1.23.3/bin/linux/amd64/kube-controller-manager -O /usr/local/bin/kube-controller-manager
  wget https://dl.k8s.io/v1.23.3/bin/linux/amd64/kube-scheduler -O /usr/local/bin/kube-scheduler
  chmod +x /usr/local/bin/kube-apiserver \
           /usr/local/bin/kube-controller-manager \
           /usr/local/bin/kube-scheduler
}
```

### Distribute TLS cerficates and configuration files
```
{
  \cp ca.key ca.crt kube-apiserver.key kube-apiserver.crt \
      sa.key sa.crt etcd-server.key etcd-server.crt \
      front-proxy-client.key front-proxy-client.crt /etc/kubernetes/pki/
  \cp kube-controller-manager.kubeconfig kube-scheduler.kubeconfig \
      encryption-config.yaml /etc/kubernetes/
}
```

### Configure the Kubernetes API Server
Each controller VM IP address will be used to advertise the API Server to members of the cluster. Retrieve the IP address for the current controller VM:
> in this example the `enp1s0` is the network interface used in my controller VMs, adjust it to match your setup
```
CONTROLLER_IP=$(ip addr show enp1s0 | grep -w "inet" | xargs | awk '{print $2}' | awk -F/ '{print $1}')
```

Create the `kube-apiserver.service` systemd unit file:
> make sure to adjust the `--etcd-servers` parameter to match your controller VMs
```
cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${CONTROLLER_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/etc/kubernetes/pki/ca.crt \\
  --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --etcd-cafile=/etc/kubernetes/pki/ca.crt \\
  --etcd-certfile=/etc/kubernetes/pki/etcd-server.crt \\
  --etcd-keyfile=/etc/kubernetes/pki/etcd-server.key \\
  --etcd-servers=https://172.19.33.11:2379,https://172.19.33.12:2379,https://172.19.33.13:2379 \\
  --event-ttl=1h \\
  --encryption-provider-config=/etc/kubernetes/encryption-config.yaml \\
  --enable-bootstrap-token-auth=true \\
  --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt \\
  --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key \\
  --requestheader-client-ca-file=/etc/kubernetes/pki/ca.crt \\
  --requestheader-allowed-names=front-proxy-client \\
  --requestheader-extra-headers-prefix=X-Remote-Extra- \\
  --requestheader-group-headers=X-Remote-Group \\
  --requestheader-username-headers=X-Remote-User \\
  --kubelet-certificate-authority=/etc/kubernetes/pki/ca.crt \\
  --kubelet-client-certificate=/etc/kubernetes/pki/kube-apiserver.crt \\
  --kubelet-client-key=/etc/kubernetes/pki/kube-apiserver.key \\
  --runtime-config='api/all=true' \\
  --service-account-key-file=/etc/kubernetes/pki/sa.crt \\
  --service-account-signing-key-file=/etc/kubernetes/pki/sa.key \\
  --service-account-issuer=https://kubernetes.default.svc.cluster.local \\
  --service-cluster-ip-range=10.96.0.0/12 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/etc/kubernetes/pki/kube-apiserver.crt \\
  --tls-private-key-file=/etc/kubernetes/pki/kube-apiserver.key \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Configure the Kubernetes Controller Manager
Create the `kube-controller-manager.service` systemd unit file:
```
cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --bind-address=0.0.0.0 \\
  --cluster-cidr=10.244.0.0/16 \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt \\
  --cluster-signing-key-file=/etc/kubernetes/pki/ca.key \\
  --kubeconfig=/etc/kubernetes/kube-controller-manager.kubeconfig \\
  --leader-elect=true \\
  --root-ca-file=/etc/kubernetes/pki/ca.crt \\
  --service-account-private-key-file=/etc/kubernetes/pki/sa.key \\
  --service-cluster-ip-range=10.96.0.0/12 \\
  --use-service-account-credentials=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Configure the Kubernetes Scheduler
Create the `kube-scheduler.service` systemd unit file:
```
cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --bind-address=0.0.0.0 \\
  --kubeconfig=/etc/kubernetes/kube-scheduler.kubeconfig \\
  --leader-elect=true
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Start the Controller Services
```
{
  systemctl daemon-reload
  systemctl enable kube-apiserver kube-controller-manager kube-scheduler
  systemctl start kube-apiserver kube-controller-manager kube-scheduler
}
```

> Allow up to 10 seconds for the Kubernetes API Server to fully initialize.

> Remember to run the above commands on each controller VM: `k8s-control-01`, `k8s-control-02`, `and k8s-control-03`.

## Verification
Check the cluster info:
```
kubectl cluster-info --kubeconfig admin.kubeconfig
```

> Expected output:
```
Kubernetes control plane is running at https://127.0.0.1:6443
```

Make an HTTP request to retrieve the Kubernetes version info:
```
curl --cacert ca.crt https://127.0.0.1:6443/version
```

> Expected output:
```
{
  "major": "1",
  "minor": "23",
  "gitVersion": "v1.23.3",
  "gitCommit": "816c97ab8cff8a1c72eccca1026f7820e93e0d25",
  "gitTreeState": "clean",
  "buildDate": "2022-01-25T21:19:12Z",
  "goVersion": "go1.17.6",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

Next: [Configuring the Kubernetes Control Plane](10-configuring-kubernetes-controllers.md)
