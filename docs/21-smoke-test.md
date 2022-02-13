# Smoke Test
In this lab, we'll complete a series of tasks to ensure the Kubernetes cluster is functioning properly.

The commands in this lab will affect the entire cluster and only need to be run once from one of the controller VMs, so run them only from the `k8s-control-01` controller VM.

## etcd Data Encryption
In this section, we'll verify the ability to [encrypt secret data at rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#verifying-that-data-is-encrypted).

Create a generic secret:
```
kubectl create secret generic kubernetes-the-hard-way \
        --from-literal="mykey=mydata"
```

Print a hexdump of the `kubernetes-the-hard-way` secret stored in etcd:
```
{
  ETCDCTL_API=3 
  etcdctl get \
          --endpoints=https://127.0.0.1:2379 \
          --cacert=/etc/etcd/pki/ca.crt \
          --cert=/etc/etcd/pki/etcd-server.crt \
          --key=/etc/etcd/pki/etcd-server.key \
          /registry/secrets/default/kubernetes-the-hard-way | hexdump -C
}
```

> Expected output:
```
00000000  2f 72 65 67 69 73 74 72  79 2f 73 65 63 72 65 74  |/registry/secret|
00000010  73 2f 64 65 66 61 75 6c  74 2f 6b 75 62 65 72 6e  |s/default/kubern|
00000020  65 74 65 73 2d 74 68 65  2d 68 61 72 64 2d 77 61  |etes-the-hard-wa|
00000030  79 0a 6b 38 73 3a 65 6e  63 3a 61 65 73 63 62 63  |y.k8s:enc:aescbc|
00000040  3a 76 31 3a 6b 65 79 31  3a 8b 60 c8 06 39 51 32  |:v1:key1:.`..9Q2|
00000050  94 67 66 98 f3 a2 88 99  a3 66 f2 2d 61 f2 93 92  |.gf......f.-a...|
00000060  7b 7d 69 55 3f 92 48 14  4c 69 72 ab f5 be fe 3f  |{}iU?.H.Lir....?|
00000070  1c c6 84 75 58 cc 1d a9  e4 8d a4 45 5b b3 bd f7  |...uX......E[...|
00000080  6c 72 a2 83 9e c4 57 b9  22 cd 02 9b 12 71 23 9b  |lr....W."....q#.|
00000090  6e 03 a4 25 97 8d ae a1  9b 8f 63 24 f7 ff 5a 5e  |n..%......c$..Z^|
000000a0  87 9f bf 6c fa 1d 94 88  e0 c2 dd 85 7b ed bc 47  |...l........{..G|
000000b0  05 99 48 8a 2d 9b 65 27  a1 64 af a4 c1 22 c1 15  |..H.-.e'.d..."..|
000000c0  a7 b9 dc ea ab cc c6 e5  04 78 20 5e c5 56 6c 0d  |.........x ^.Vl.|
000000d0  84 06 e0 53 17 70 72 2a  25 9f 82 25 15 67 d5 01  |...S.pr*%..%.g..|
000000e0  80 3e 33 cb a4 ae 28 90  01 be b9 ef 6f b9 40 4d  |.>3...(.....o.@M|
000000f0  95 d3 a5 5e 09 69 85 69  d3 c6 69 5e b8 eb b7 fc  |...^.i.i..i^....|
00000100  39 17 22 ec 37 e7 2f 82  ff 43 cd 36 95 56 72 7d  |9.".7./..C.6.Vr}|
00000110  d4 a3 53 ad 3a 40 6a 7d  bb 57 70 77 f4 58 2f 55  |..S.:@j}.Wpw.X/U|
00000120  95 27 e8 5a e1 25 2d fa  71 b4 dc 4a 77 30 34 ef  |.'.Z.%-.q..Jw04.|
00000130  6a 22 a5 2c a6 45 8f 6e  59 0e a9 9c b5 d9 42 dc  |j".,.E.nY.....B.|
00000140  82 0e 1c a5 bc 49 14 f6  6a 6f e8 1c 05 4e be 15  |.....I..jo...N..|
00000150  35 62 09 d3 d1 89 9e e6  a6 0a                    |5b........|
0000015a
```
> The etcd key should be prefixed with `k8s:enc:aescbc:v1:key1`, which indicates the `aescbc` provider was used to encrypt the data with the `key1` encryption key.

## Deployments
In this section, we'll verify the ability to create and manage [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).

Create a deployment for the [nginx](https://nginx.org/en/) web server:
```
kubectl create deployment nginx --image=nginx
```

List the Pods created by the `nginx` deployment:
```
kubectl get pods -l app=nginx
```
> Expected output:
```
NAME                     READY   STATUS    RESTARTS   AGE
nginx-85b98978db-bz8xk   1/1     Running   0          30s
```

## Port Forwarding
In this section, we'll verify the ability to access applications remotely using [port forwarding](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/).

Retrieve the full name of the `nginx` pod:
```
POD_NAME=$(kubectl get pods -l app=nginx -o jsonpath="{.items[0].metadata.name}")
```

Forward port `8080` on your local machine to port `80` of the `nginx` pod:
```
kubectl port-forward $POD_NAME 8080:80
```
> Expected output:
```
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

In a new terminal make an HTTP request using the forwarding address:
```
curl --head http://127.0.0.1:8080
```
> Expected output:
```
HTTP/1.1 200 OK
Server: nginx/1.21.6
Date: Sat, 12 Feb 2022 21:48:48 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Tue, 25 Jan 2022 15:03:52 GMT
Connection: keep-alive
ETag: "61f01158-267"
Accept-Ranges: bytes
```

Switch back to the previous terminal and stop the port forwarding to the `nginx` pod by pressing the ctrl+c key combination.

## Logs
In this section, we'll verify the ability to [retrieve container logs](https://kubernetes.io/docs/concepts/cluster-administration/logging/).

Print the `nginx` Pod logs:
```
kubectl logs ${POD_NAME}
```
> Expected output:
```
...
127.0.0.1 - - [12/Feb/2022:21:48:48 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.61.1" "-"
...
```

## Running commands in a container
In this section, we'll verify the ability to [execute commands in a container](https://kubernetes.io/docs/tasks/debug-application-cluster/get-shell-running-container/#running-individual-commands-in-a-container).

Print the nginx version by executing the `nginx -v` command in the `nginx` container:
```
kubectl exec -ti ${POD_NAME} -- nginx -v
```
> Expected output:
```
nginx version: nginx/1.21.6
```

## Services
In this section, we'll verify the ability to expose applications using a [Service](https://kubernetes.io/docs/concepts/services-networking/service/).

Expose the `nginx` deployment using a [NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport) service:
```
kubectl expose deployment nginx --port 80 --type NodePort
```

Retrieve the node port assigned to the `nginx` service:
```
kubectl get services nginx
```
> Expected output:
```
NAME    TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
nginx   NodePort   10.101.12.198   <none>        80:30291/TCP   10s
```
> From the above ouput, the nginx Service was assigned with `30291` port.

From the KVM host (`k8s-kvm-host`), make an HTTP request to IP address of one of the controller or worker VMs and the `nginx` NodePort (for example `k8s-control-01`):
```
curl -I http://172.19.33.11:30291
```

> Expected output:
```
HTTP/1.1 200 OK
Server: nginx/1.21.6
Date: Sat, 12 Feb 2022 21:59:29 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Tue, 25 Jan 2022 15:03:52 GMT
Connection: keep-alive
ETag: "61f01158-267"
Accept-Ranges: bytes
```

## Clean up
Remove the resources created in this lab:
```
{
  kubectl delete services nginx
  kubectl delete deployments nginx
  kubectl delete secrets kubernetes-the-hard-way
}
```

Next: [Cleaning Up](22-cleanup.md)
