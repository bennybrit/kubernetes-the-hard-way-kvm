# Deploying the LoadBalancer Service Add-on

In this lab, we'll deploy a [LoadBalancer Service add-on](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer) which will allow us to expose Service externally.

Kubernetes does not offer an implementation of network load balancers (Services of type LoadBalancer) for bare-metal clusters, so we'll be using the [MetalLB](https://metallb.universe.tf/) add-on to add this capability to our cluster.

The commands in this lab will affect the entire cluster and only need to be run once from one of the controller VMs, so run them only from the `k8s-control-01` controller VM.

## The LoadBalancer Service Add-on
Deploy the MetalLB (v0.11.0) add-on:
```
kubectl apply -f https://raw.githubusercontent.com/bennybrit/kubernetes-the-hard-way-kvm/main/deployments/metallb-0.11.0.yaml
```

> Expected output:
```
namespace/metallb-system created
podsecuritypolicy.policy/controller created
podsecuritypolicy.policy/speaker created
serviceaccount/controller created
serviceaccount/speaker created
clusterrole.rbac.authorization.k8s.io/metallb-system:controller created
clusterrole.rbac.authorization.k8s.io/metallb-system:speaker created
role.rbac.authorization.k8s.io/config-watcher created
role.rbac.authorization.k8s.io/pod-lister created
role.rbac.authorization.k8s.io/controller created
clusterrolebinding.rbac.authorization.k8s.io/metallb-system:controller created
clusterrolebinding.rbac.authorization.k8s.io/metallb-system:speaker created
rolebinding.rbac.authorization.k8s.io/config-watcher created
rolebinding.rbac.authorization.k8s.io/pod-lister created
rolebinding.rbac.authorization.k8s.io/controller created
daemonset.apps/speaker created
deployment.apps/controller created
```

Layer 2 mode is the simplest way to configure MetalLB, for this mode, you don’t need any protocol-specific configuration, only IP addresses allocation, Generate a ConfigMap object with the ranges of IPs which will be used by the MetalLB add-on (make sure to adjust the `addresses` proprty to match your network topology):
```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
      - name: default
        protocol: layer2
        addresses:
          - 172.19.33.50-172.19.33.60
EOF
```
> In the above ConfigMap i've allocated the IP addresses in the range of `172.19.33.50-172.19.33.60`

List the pods created by the `MetalLB` deployment:
```
kubectl get pods -n metallb-system -o wide
```

> Expected output:
```
NAME                          READY   STATUS    RESTARTS   AGE     IP             NODE             NOMINATED NODE   READINESS GATES
controller-7cf77c64fb-fpqpk   1/1     Running   0          30s     10.36.0.3      k8s-worker-01    <none>           <none>
speaker-n2jwl                 1/1     Running   0          30s     172.19.33.11   k8s-control-01   <none>           <none>
speaker-628j4                 1/1     Running   0          30s     172.19.33.12   k8s-control-02   <none>           <none>
speaker-nhbgw                 1/1     Running   0          30s     172.19.33.13   k8s-control-03   <none>           <none>
speaker-45qrj                 1/1     Running   0          30s     172.19.33.14   k8s-worker-01    <none>           <none>
speaker-gz446                 1/1     Running   0          30s     172.19.33.15   k8s-worker-02    <none>           <none>
speaker-tpqmz                 1/1     Running   0          30s     172.19.33.16   k8s-worker-03    <none>           <none>
```

## Verification
Deploy a simple nginx Deployment together with LoadBalancer Service
```
kubectl apply -f https://raw.githubusercontent.com/bennybrit/kubernetes-the-hard-way-kvm/main/deployments/metallb-e2e-test.yaml
```

> Expected output:
```
deployment.apps/nginx created
service/nginx-lb created
```

List the Pods and Service that have been created:
```
kubectl get service,pods -l k8s-app=metallb-e2e
```

> Expected output:
```
NAME               TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)        AGE
service/nginx-lb   LoadBalancer   10.99.138.95   172.19.33.50   80:32266/TCP   100s

NAME                        READY   STATUS    RESTARTS   AGE
pod/nginx-f44697c6c-j9j8k   1/1     Running   0          100s
pod/nginx-f44697c6c-p8jgp   1/1     Running   0          100s
pod/nginx-f44697c6c-t7qs8   1/1     Running   0          100s
```
> From the above output, we can see that the `nginx-lb` Service was assigned with the `172.19.33.50` external IP address.

Try to access the Service from an external server (outside the Kubernetes cluster), for example you can run it on the KVM host (`k8s-kvm-host`):
```
curl 172.19.33.50
```

> Expected output:
```
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>
```

Clean-up the reosurces used for the verification:
```
{
  kubectl delete deployments nginx
  kubectl delete service nginx-lb
}
```

Next: [Deploying the Ingress Controller](18-ingress-controller.md)
