# Installing Container runtime

The Container runtime will be used by Kubernetes to run containers.

For the Container runtime, we'll be using Docker Engine (v20.10.12).

The Container runtime should be installed on the controller VMs (`k8s-control-01`, `k8s-control-02`, `k8s-control-03`) and worker VMs (`k8s-worker-01`, `k8s-worker-02`, `k8s-worker-03`), so make sure you run the upcoming steps on each one of them.

## Install Container runtime (Docker)

Add the stable docker yum repository:
```
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```
Install Docker Engine (20.10.12) and containerd (1.4.12):
```
yum -y install docker-ce-20.10.12 docker-ce-cli-20.10.12 containerd.io-1.4.12
```
Create required directories:
```
mkdir -p /etc/docker
```
Set the cgroup driver to systemd:
```
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
```
Enable and start docker service:
```
{
  systemctl enable docker.service
  systemctl start docker.service
}
```

## Verification
Validate docker is up and running by running the "hello world" example container:
```
docker run hello-world
```
> Expected output:
```
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

Validate that IP forwarding is enabled:
```
cat /proc/sys/net/ipv4/ip_forward
```
> Expected output:
```
1
```

Validate that "overlay" and "br_netfilter" kernel modules were loaded:
```
lsmod | grep 'overlay\|br_netfilter'
```
> Expected output:
```
br_netfilter           24576  0
overlay               139264  0
```

Next: [Installing the kubectl Client Tool](04-client-tools.md)
