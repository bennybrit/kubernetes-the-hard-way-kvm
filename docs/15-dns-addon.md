# Deploying the DNS Cluster Add-on

In this lab, we'll deploy the [DNS add-on](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/) which provides DNS based service discovery, backed by [CoreDNS](https://coredns.io/), to applications running inside the Kubernetes cluster.

The commands in this lab will affect the entire cluster and only need to be run once from one of the controller VMs, so run them only from the `k8s-control-01` controller VM.

## The DNS Cluster Add-on

Deploy the CoreDNS (v1.8.7) cluster add-on:
```
kubectl apply -f https://raw.githubusercontent.com/bennybrit/kubernetes-the-hard-way/master/deployments/coredns-1.8.7.yaml
```

> Expected output:
```
serviceaccount/coredns created
clusterrole.rbac.authorization.k8s.io/system:coredns created
clusterrolebinding.rbac.authorization.k8s.io/system:coredns created
configmap/coredns created
deployment.apps/coredns created
service/kube-dns created
```

List the pods created by the `kube-dns` deployment:
```
kubectl get pods -l k8s-app=kube-dns -n kube-system -o wide
```

> Expected output:
```
NAME                      READY   STATUS    RESTARTS   AGE   IP          NODE            NOMINATED NODE   READINESS GATES
coredns-68764ddc4-qj9b7   1/1     Running   0          15s   10.36.0.1   k8s-worker-01   <none>           <none>
coredns-68764ddc4-zfg8x   1/1     Running   0          15s   10.39.0.1   k8s-worker-02   <none>           <none>
coredns-68764ddc4-hhhlj   1/1     Running   0          15s   10.44.0.1   k8s-worker-03   <none>           <none>
```

## Verification
Create a `dnsutils` Pod:
```
kubectl run dnsutils --image=k8s.gcr.io/e2e-test-images/jessie-dnsutils:1.3 --command -- sleep 3600
```

List the `dnsutils` Pod created:
```
kubectl get pods -l run=dnsutils
```

> Expected output:
```
NAME       READY   STATUS    RESTARTS   AGE
dnsutils   1/1     Running   0          30s
```

Execute a DNS lookup for the `kubernetes` service inside the `dnsutils` Pod:
```
kubectl exec -ti dnsutils -- nslookup kubernetes
```

> Expected output:
```
Server:		10.96.0.10
Address:	10.96.0.10#53

Name:	kubernetes.default.svc.cluster.local
Address: 10.96.0.1
```

Remove the `dnsutils` Pod:
```
kubectl delete pod dnsutils
```

Next: [Deploying the Metrics Server Add-on](16-metrics-server-addon.md)
