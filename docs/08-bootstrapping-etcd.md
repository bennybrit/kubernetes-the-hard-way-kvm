# Bootstrapping the etcd Cluster

Kubernetes components are stateless and store cluster state in [etcd](https://github.com/etcd-io/etcd). In this lab we'll bootstrap a three node etcd cluster and configure it for high availability and secure remote access.

etcd will be installed on the controller VMs (`k8s-control-01`, `k8s-control-02`, and `k8s-control-03`), so make sure you run the upcoming steps on each one.

## Bootstrapping an etcd cluster member

### Download and install the etcd binaries
Download the official etcd release binaries (v3.5.2) from the [etcd](https://github.com/etcd-io/etcd) GitHub project:
```
wget -q --show-progress --https-only --timestamping \
  "https://github.com/etcd-io/etcd/releases/download/v3.5.2/etcd-v3.5.2-linux-amd64.tar.gz"
```

Extract and install the `etcd` server and the `etcdctl` command line utility:
```
{
  tar -xvf etcd-v3.5.2-linux-amd64.tar.gz
  \mv etcd-v3.5.2-linux-amd64/etcd* /usr/local/bin/
  chown root:root /usr/local/bin/etcd*
  rm -rf etcd-v3.5.2-linux-amd64*
}
```

### Configure the etcd server
Create the required directories, and distribute the required keys and certificates:
```
{
  mkdir -p /etc/etcd/pki /var/lib/etcd
  chmod 700 /var/lib/etcd
  \cp ca.crt etcd-server.key etcd-server.crt /etc/etcd/pki/
}
```

Each controller VM IP address will be used to serve client requests and communicate with etcd cluster peers. Retrieve the IP address for the current controller VM:
> in this example the `enp1s0` is the network interface used in my controller VMs, adjust it to match your setup
```
ETCD_IP=$(ip addr show enp1s0 | grep -w "inet" | xargs | awk '{print $2}' | awk -F/ '{print $1}')
```

Each etcd member must have a unique name within an etcd cluster. Set the etcd name to match the hostname of the current controller VM:
```
ETCD_NAME=$(hostname -s)
```

Create the `etcd.service` systemd unit file:
> make sure to adjust the `--initial-cluster` parameter to match all your controller VMs

```
cat <<EOF | tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/pki/etcd-server.crt \\
  --key-file=/etc/etcd/pki/etcd-server.key \\
  --peer-cert-file=/etc/etcd/pki/etcd-server.crt \\
  --peer-key-file=/etc/etcd/pki/etcd-server.key \\
  --trusted-ca-file=/etc/etcd/pki/ca.crt \\
  --peer-trusted-ca-file=/etc/etcd/pki/ca.crt \\
  --peer-client-cert-auth=true \\
  --client-cert-auth=true \\
  --initial-advertise-peer-urls https://${ETCD_IP}:2380 \\
  --listen-peer-urls https://${ETCD_IP}:2380 \\
  --listen-client-urls https://${ETCD_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${ETCD_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster k8s-control-01=https://172.19.33.11:2380,k8s-control-02=https://172.19.33.12:2380,k8s-control-03=https://172.19.33.13:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Enable and start the etcd server
```
{
  systemctl daemon-reload
  systemctl enable etcd
  systemctl start etcd
}
```

> Remember to run the above commands on each controller VM: `k8s-control-01`, `k8s-control-02`, and `k8s-control-03`.

## Verification
List the etcd cluster members:
```
ETCDCTL_API=3
etcdctl member list \
        --endpoints=https://127.0.0.1:2379 \
        --cacert=/etc/etcd/pki/ca.crt \
        --cert=/etc/etcd/pki/etcd-server.crt \
        --key=/etc/etcd/pki/etcd-server.key
```

> Expected output:
```
60b3627eeb53fca2, started, k8s-control-01, https://172.19.33.11:2380, https://172.19.33.11:2379, false
a9edc46a6e38d3d3, started, k8s-control-02, https://172.19.33.12:2380, https://172.19.33.12:2379, false
84e172b6bd7c6b70, started, k8s-control-03, https://172.19.33.13:2380, https://172.19.33.13:2379, false
```

Next: [Bootstrapping the Kubernetes Control Plane](09-bootstrapping-kubernetes-controllers.md)
