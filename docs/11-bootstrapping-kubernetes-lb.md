# Bootstrapping the Kubernetes API Load Balancer

In this lab, we'll provision an external load balancer to front the Kubernetes API Servers. We'll be using [HAProxy](http://www.haproxy.org/) as the load balancer solution.

The HAProxy load balancer will be installed and configured on the Load Balancer VM (`k8s-lb-01`), so make sure you run the upcoming steps only on it.

## Install and configure HAProxy Load Balancer
Install the HAProxy Load Balancer package:
```
yum -y install haproxy
```

The Load Balancer VM IP address will be used as the frontend for the Kubernetes API servers. Retrieve the IP address:
> in this example the `enp1s0` is the network interface used in my Load Balancer VM, adjust it to match your setup
```
LOADBALANCER_IP=$(ip addr show enp1s0 | grep -w "inet" | xargs | awk '{print $2}' | awk -F/ '{print $1}')
```

Generate the load balancer configuration:
> In case your controller VMs (`k8s-control-01`, `k8s-control-02`, and `k8s-control-03`) are configured with different IP addresses make sure to adjust them under the `backend k8s_apiservers` section
```
cat <<EOF | sudo tee /etc/haproxy/haproxy.cfg 
frontend apiservers
    bind ${LOADBALANCER_IP}:6443
    mode tcp
    option tcplog
    option forwardfor
    default_backend k8s_apiservers    

backend k8s_apiservers
    mode tcp
    option tcplog
    balance roundrobin
    server k8s-control-01 172.19.33.11:6443 check fall 3 rise 2
    server k8s-control-02 172.19.33.12:6443 check fall 3 rise 2
    server k8s-control-03 172.19.33.13:6443 check fall 3 rise 2

listen stats
   bind ${LOADBALANCER_IP}:80
   mode http
   stats enable
   stats uri /
EOF
```

Enable and start the `haproxy` service:
```
{
  systemctl enable haproxy.service 
  systemctl start haproxy.service
}
```

## Verification
Run the verification steps from the KVM host (`k8s-kvm-host`)

Make an HTTP request toward the load balancer to retrieve the Kubernetes version info:
```
curl --cacert ca.crt https://172.19.33.10:6443/version
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

> You can also access the load balancer stats webpage (`http://172.19.33.10/`), and inspect the different statistics.

Next: [Bootstrapping the Kubernetes Worker Nodes](12-bootstrapping-kubernetes-workers.md)
