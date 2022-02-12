# Deploying the Ingress Controller

In this lab, we'll deploy an [Ingress Controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/) which will allow us to create [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) resources.

We'll be using the [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/deploy/).

The commands in this lab will affect the entire cluster and only need to be run once from one of the controller VMs, so run them only from the `k8s-control-01` controller VM.

## The Ingress Controller
Deploy the NGINX Ingress Controller (v1.1.1) add-on:
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.1.1/deploy/static/provider/cloud/deploy.yaml
```

> Expected output:
```
namespace/ingress-nginx created
serviceaccount/ingress-nginx created
configmap/ingress-nginx-controller created
clusterrole.rbac.authorization.k8s.io/ingress-nginx created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx created
role.rbac.authorization.k8s.io/ingress-nginx created
rolebinding.rbac.authorization.k8s.io/ingress-nginx created
service/ingress-nginx-controller-admission created
service/ingress-nginx-controller created
deployment.apps/ingress-nginx-controller created
ingressclass.networking.k8s.io/nginx created
validatingwebhookconfiguration.admissionregistration.k8s.io/ingress-nginx-admission created
serviceaccount/ingress-nginx-admission created
clusterrole.rbac.authorization.k8s.io/ingress-nginx-admission created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
role.rbac.authorization.k8s.io/ingress-nginx-admission created
rolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
job.batch/ingress-nginx-admission-create created
job.batch/ingress-nginx-admission-patch created
```

List the pods created by the `ingress-nginx-controller` deployment:
```
kubectl get pods -n ingress-nginx
```

> Expected output:
```
NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-f4v7j        0/1     Completed   0          90s
ingress-nginx-admission-patch-8zlnl         0/1     Completed   0          90s
ingress-nginx-controller-568764d844-w7kpv   1/1     Running     0          90s
```

## Verification
The following deployment will deploy nginx webserver Deployment, apache webserver Deployment, ClusterIP Service for each Deployment, and an Ingress resource which will provide access to each webserver by using a unique path (`/nginx` for the nginx webserver and `/apache` for the apache webserver):
```
kubectl apply -f https://raw.githubusercontent.com/bennybrit/kubernetes-the-hard-way/master/deployments/ingress-e2e-test.yaml
```

> Expected output:
```
deployment.apps/nginx created
service/nginx-clusterip created
deployment.apps/apache created
service/apache-clusterip created
ingress.networking.k8s.io/ingress-e2e-test created
```

List the resources that have been created:
```
kubectl get deployments,service,ingress -l k8s-app=ingress-e2e
```

> Expected output:
```
NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/apache   3/3     3            3           30s
deployment.apps/nginx    3/3     3            3           30s

NAME                       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/apache-clusterip   ClusterIP   10.103.199.232   <none>        8080/TCP   30s
service/nginx-clusterip    ClusterIP   10.98.89.120     <none>        8080/TCP   30s

NAME                                         CLASS   HOSTS   ADDRESS        PORTS   AGE
ingress.networking.k8s.io/ingress-e2e-test   nginx   *       172.19.33.50   80      30s
```
> From the above output, we can see that the `ingress-e2e-test` Ingress resource was assigned with the `172.19.33.50` external IP address.

Try to access the different webservers from an external server using each unique path (outside the Kubernetes cluster), for example, you can run it on the KVM host (`k8s-kvm-host`).

First try to access the apache webserver:
```
curl 172.19.33.50/apache
```

> Expected output:
```
<html><body><h1>It works!</h1></body></html>
```

Now try to access the nginx webserver:
```
curl 172.19.33.50/nginx
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
  kubectl delete service apache-clusterip 
  kubectl delete service nginx-clusterip 
  kubectl delete deployments apache 
  kubectl delete deployments nginx
  kubectl delete ingress ingress-e2e-test
}
```

Next: [Bootstrapping the NFS Server](19-bootstrapping-nfs-server.md)
