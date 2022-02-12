# Configuring the Kubernetes Control Plane

In this lab, we'll extend the Kubernetes control plane configuration.

First, we'll configure RBAC permissions to allow the Kubernetes API Server to access the Kubelet API on each worker node, then we'll configure TLS client certificate bootstrapping for kubelet which will allow us easily add the worker nodes to the cluster.

The commands in this lab will affect the entire cluster and only need to be run once from one of the controller VMs, so run them only from the `k8s-control-01` controller VM.

## RBAC for Kubelet Authorization
In this section, we'll configure RBAC permissions to allow the Kubernetes API Server to access the Kubelet API on each worker node. Access to the Kubelet API is required for retrieving metrics, logs, and executing commands in pods.

Create the `system:kube-apiserver-to-kubelet` [ClusterRole](https://kubernetes.io/docs/admin/authorization/rbac/#role-and-clusterrole) with permissions to access the Kubelet API and perform most common tasks associated with managing pods:
```
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - proxy
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF
```

The Kubernetes API Server authenticates to the Kubelet as the `kube-apiserver` user using the client certificate as defined by the `--kubelet-client-certificate` flag.

Bind the `system:kube-apiserver-to-kubelet` ClusterRole to the `kube-apiserver` user:

```
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kube-apiserver
EOF
```

## TLS client certificate bootstrapping for kubelets
In this section, we'll set up [TLS client certificate bootstrapping for kubelets](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet-tls-bootstrapping/) which will allow us to add the different worker nodes to the cluster using a certificate request and signing API.

### Generate Bootstrap Token Secret
In order for the bootstrapping kubelet to connect to kube-apiserver and request a certificate, it must first authenticate to the server, for this will use bootstrap token which will be stored as a Secret object.

Generate the bootstrap token Secret object:
> Make sure to set the `expiration` property to a date in the future
```
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: v1
kind: Secret
metadata:
  name: bootstrap-token-07401b
  namespace: kube-system
type: bootstrap.kubernetes.io/token
stringData:
  description: "The default bootstrap token for k8s workers"
  token-id: 07401b
  token-secret: f395accd246ae52d
  expiration: 2025-01-01T00:00:00Z
  usage-bootstrap-authentication: "true"
  usage-bootstrap-signing: "true"
  auth-extra-groups: system:bootstrappers:worker
EOF
```

> The token is constructed from the `token-id` and `token-secret` properties, from the above example, the token will be `07401b.f395accd246ae52d`, write it down as it will be used while [Bootstrapping the Kubernetes Worker Nodes](12-bootstrapping-kubernetes-workers.md)

### Authorize kubelet to create CSR
Create a ClusterRoleBinding which will allow kubelet to create a certificate signing request (CSR) as well as retrieve it when done:
```
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: create-csrs-for-bootstrapping
subjects:
- kind: Group
  name: system:bootstrappers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: system:node-bootstrapper
  apiGroup: rbac.authorization.k8s.io
EOF
```

### Authorize controller-manager to approve CSR
Create a ClusterRoleBinding which will allow kubelet to request and receive a new certificate:
```
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: auto-approve-csrs-for-group
subjects:
- kind: Group
  name: system:bootstrappers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: system:certificates.k8s.io:certificatesigningrequests:nodeclient
  apiGroup: rbac.authorization.k8s.io
EOF
```

### Authorize kubelet to renew a certificate
Create a ClusterRoleBinding which will allow kubelet to renew its own client certificate:
```
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: auto-approve-renewals-for-nodes
subjects:
- kind: Group
  name: system:nodes
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: system:certificates.k8s.io:certificatesigningrequests:selfnodeclient
  apiGroup: rbac.authorization.k8s.io
EOF
```

Next: [Bootstrapping the Kubernetes API Load Balancer](11-bootstrapping-kubernetes-lb.md)
