# Deploying the Metrics Server Add-on

In this lab, we'll deploy a [tool for monitoring cluster resources](https://kubernetes.io/docs/tasks/debug-application-cluster/resource-usage-monitoring/) which will provide us with resource metrics for the different Kubernetes cluster resources (Pods and Nodes).

We'll deploy the [Kubernetes Metrics Server](https://github.com/kubernetes-sigs/metrics-server) in high availability mode.

The commands in this lab will affect the entire cluster and only need to be run once from one of the controller VMs, so run them only from the `k8s-control-01` controller VM.

## The Metrics Server Add-on
Deploy the Metrics Server (v0.6.1) add-on:
```
kubectl apply -f https://raw.githubusercontent.com/bennybrit/kubernetes-the-hard-way/master/deployments/metrics-server-0.6.1.yaml
```

> Expected output:
```
serviceaccount/metrics-server created
clusterrole.rbac.authorization.k8s.io/system:aggregated-metrics-reader created
clusterrole.rbac.authorization.k8s.io/system:metrics-server created
rolebinding.rbac.authorization.k8s.io/metrics-server-auth-reader created
clusterrolebinding.rbac.authorization.k8s.io/metrics-server:system:auth-delegator unchanged
clusterrolebinding.rbac.authorization.k8s.io/system:metrics-server created
service/metrics-server created
deployment.apps/metrics-server created
poddisruptionbudget.policy/metrics-server created
apiservice.apiregistration.k8s.io/v1beta1.metrics.k8s.io created
```

List the pods created by the `metrics-server` deployment:
```
kubectl get pods -l k8s-app=metrics-server -n kube-system -o wide
```

> Expected output:
```
NAME                              READY   STATUS    RESTARTS   AGE   IP          NODE            NOMINATED NODE   READINESS GATES
metrics-server-5bffb474fc-7wjx7   1/1     Running   0          30s   10.36.0.2   k8s-worker-01   <none>           <none>
metrics-server-5bffb474fc-m5rlg   1/1     Running   0          30s   10.39.0.2   k8s-worker-02   <none>           <none>
metrics-server-5bffb474fc-mfzjv   1/1     Running   0          30s   10.44.0.2   k8s-worker-03   <none>           <none>
```

> Allow up to 30 seconds for the Metrics Server to fully initialize.

## Verification
Check the resource consumption of the different Nodes:
```
kubectl top node
```

> Expected output:
```
NAME             CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
k8s-control-01   100m         1%     1265Mi          15%       
k8s-control-02   100m         1%     1265Mi          15%       
k8s-control-03   100m         1%     1265Mi          15%       
k8s-worker-01    45m          0%     825Mi           10%       
k8s-worker-02    45m          0%     825Mi           10%       
k8s-worker-03    45m          0%     825Mi           10%   
```

Next: [Deploying the LoadBalancer Service Add-on](17-loadbalancer-service-addon.md)
