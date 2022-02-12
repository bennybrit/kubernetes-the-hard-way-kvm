# Provisioning a CA and Generating TLS Certificates

In this lab, we'll provision a [PKI Infrastructure](https://en.wikipedia.org/wiki/Public_key_infrastructure) using openssl toolkit, then use it to bootstrap a Certificate Authority, and generate TLS certificates for the following components: `etcd`, `kube-apiserver`, `kube-controller-manager`, `kube-scheduler`, and `kube-proxy`.

We'll generate the different certificates and keys on the KVM host (`k8s-kvm-host`), and from it, we'll distribute them to the different Kubernetes cluster VMs.

## Certificate Authority (CA)
In this section, you will provision a Certificate Authority that can be used to generate the different TLS certificates for the Kubernetes cluster components.

Generate the Certificate Authority (CA) private key, CSR, and then self sign:
```
{
  openssl genrsa -out ca.key 2048
  openssl req -new -key ca.key -subj "/CN=Kubernetes-CA" -out ca.csr
  openssl x509 -req -in ca.csr -signkey ca.key -CAcreateserial -days 3650 -out ca.crt
}
```
> The following files will be created: `ca.key`, `ca.csr` and `ca.crt`

## Client and Server Certificates

In this section, we'll generate client and server certificates for the different Kubernetes components and a client certificate for the Kubernetes "admin" user.

### The Admin Client Certificate
The admin client certificate will be used to access the Kubernetes API (using the kubectl tool).

Generate the "admin" client private key, CSR (we'll generate it with **"kubernetes-admin"** CN and **"system:masters"** group), and sign it using the CA private key:
```
{
  openssl genrsa -out admin.key 2048
  openssl req -new -key admin.key -subj "/CN=kubernetes-admin/O=system:masters" -out admin.csr
  openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key -CAcreateserial -days 3650 -out admin.crt
}
```
> The following files will be created: `admin.key`, `admin.csr` and `admin.crt`

### The Controller Manager Client Certificate
Generate the Controller Manager client private key, CSR (we'll generate it with **"system:kube-controller-manager"** CN), and sign it using the CA private key:
```
{
  openssl genrsa -out kube-controller-manager.key 2048
  openssl req -new -key kube-controller-manager.key -subj "/CN=system:kube-controller-manager" -out kube-controller-manager.csr
  openssl x509 -req -in kube-controller-manager.csr -CA ca.crt -CAkey ca.key -CAcreateserial -days 3650 -out kube-controller-manager.crt
}
```
> The following files will be created: `kube-controller-manager.key`, `kube-controller-manager.csr` and `kube-controller-manager.crt`

### The Scheduler Client Certificate
Generate the Scheduler client private key, CSR (we'll generate it with **"system:kube-scheduler"** CN), and sign it using the CA private key:
```
{
  openssl genrsa -out kube-scheduler.key 2048
  openssl req -new -key kube-scheduler.key -subj "/CN=system:kube-scheduler" -out kube-scheduler.csr
  openssl x509 -req -in kube-scheduler.csr -CA ca.crt -CAkey ca.key -CAcreateserial -days 3650 -out kube-scheduler.crt
}
```
> The following files will be created: `kube-scheduler.key`, `kube-scheduler.csr` and `kube-scheduler.crt`

### The Kube Proxy Client Certificate
Generate the Kube Proxy client private key, CSR (we'll generate it with **"system:kube-proxy"** CN), and sign it using the CA private key:
```
{
  openssl genrsa -out kube-proxy.key 2048
  openssl req -new -key kube-proxy.key -subj "/CN=system:kube-proxy" -out kube-proxy.csr
  openssl x509 -req -in kube-proxy.csr -CA ca.crt -CAkey ca.key -CAcreateserial -days 3650 -out kube-proxy.crt
}
```
> The following files will be created: `kube-proxy.key`, `kube-proxy.csr` and `kube-proxy.crt`

### The Front Proxy Client Certificate
Generate the Front Proxy client private key, CSR (we'll generate it with **"front-proxy-client"** CN), and sign it using the CA private key:
```
{
  openssl genrsa -out front-proxy-client.key 2048
  openssl req -new -key front-proxy-client.key -subj "/CN=front-proxy-client" -out front-proxy-client.csr
  openssl x509 -req -in front-proxy-client.csr -CA ca.crt -CAkey ca.key -CAcreateserial -days 3650 -out front-proxy-client.crt
}
```
> The following files will be created: `front-proxy-client.key`, `front-proxy-client.csr` and `front-proxy-client.crt`

### The Kubernetes API Server Certificate
For the Kubernetes API Server certificate, we'll need to include all the possible DNS names and IP addresses that the Kubernetes API Server can be reached at, this includes the IPs of all the controller VMs, load balancer VM, and localhost.

To include all the different DNS names and IP addresses, we'll generate a conf file which then will be used when generating the CSR (you may need to adjust the IPs to match your network topology):
```
cat > kube-apiserver.cnf <<EOF
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[ dn ]
CN = kube-apiserver

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster
DNS.5 = kubernetes.default.svc.cluster.local
DNS.6 = k8s-lb-01
DNS.7 = k8s-control-01
DNS.8 = k8s-control-02
DNS.9 = k8s-control-03
IP.1 = 172.19.33.10
IP.2 = 172.19.33.11
IP.3 = 172.19.33.12
IP.4 = 172.19.33.13
IP.5 = 10.96.0.1
IP.6 = 127.0.0.1

[ v3_ext ]
authorityKeyIdentifier=keyid,issuer:always
basicConstraints=CA:FALSE
keyUsage=keyEncipherment,dataEncipherment
extendedKeyUsage=serverAuth,clientAuth
subjectAltName=@alt_names
EOF
```
> In the above conf file, I've included the `k8s-lb-01`, `k8s-control-01`, `k8s-control-02`, and `k8s-control-03` DNS names and their corresponding IP addresses (`172.19.33.10`, `172.19.33.11`, `172.19.33.12`, and `172.19.33.13`).
>  
>  The Kubernetes API server is automatically assigned the `kubernetes` internal DNS name, which will be linked to the first IP address (`10.96.0.1`) from the address range (`10.96.0.0/12`) reserved for internal cluster services during the [control plane bootstrapping](09-bootstrapping-kubernetes-controllers.md) lab.

Generate the Kubernetes API Server private key, CSR (we'll use the conf file created above), and sign it using the CA private key:
```
{
  openssl genrsa -out kube-apiserver.key 2048
  openssl req -new -key kube-apiserver.key -config kube-apiserver.cnf -out kube-apiserver.csr
  openssl x509 -req -in kube-apiserver.csr -CA ca.crt -CAkey ca.key -CAcreateserial -days 3650 \
          -extensions v3_ext -extfile kube-apiserver.cnf -out kube-apiserver.crt
}
```
> The following files will be created: `kube-apiserver.key`, `kube-apiserver.csr` and `kube-apiserver.crt`

### The ETCD Server Certificate
For the ETCD Server certificate, we'll need to include all the possible IP addresses that the ETCD Server can be reached at, this includes the IPs of all the controller VMs and localhost.

To include all the different IP addresses, we'll generate a conf file which then will be used when generating the CSR (you may need to adjust the IPs to match your network topology):
```
cat > etcd-server.cnf <<EOF
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[ dn ]
CN = etcd-server

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = k8s-control-01
DNS.2 = k8s-control-02
DNS.3 = k8s-control-03
IP.1 = 172.19.33.11
IP.2 = 172.19.33.12
IP.3 = 172.19.33.13
IP.4 = 127.0.0.1

[ v3_ext ]
authorityKeyIdentifier=keyid,issuer:always
basicConstraints=CA:FALSE
keyUsage=keyEncipherment,dataEncipherment
extendedKeyUsage=serverAuth,clientAuth
subjectAltName=@alt_names
EOF
```
> In the above conf file, I've included the `k8s-control-01`, `k8s-control-02`, and `k8s-control-03` DNS names and their corresponding IP addresses (`172.19.33.11`, `172.19.33.12`, and `172.19.33.13`).

Generate the Kubernetes API Server private key, CSR (we'll use the conf file created above), and sign it using the CA private key:
```
{
  openssl genrsa -out etcd-server.key 2048
  openssl req -new -key etcd-server.key -config etcd-server.cnf -out etcd-server.csr
  openssl x509 -req -in etcd-server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -days 3650 \
          -extensions v3_ext -extfile etcd-server.cnf -out etcd-server.crt
}
```
> The following files will be created: `etcd-server.key`, `etcd-server.csr` and `etcd-server.crt`

## The Service Account Key Pair
The Kubernetes Controller Manager leverages a key pair to generate and sign service account tokens as described in the [managing service accounts](https://kubernetes.io/docs/admin/service-accounts-admin/) documentation.

Generate the service-account private key, CSR (we'll generate it with **"service-accounts"** CN), and sign it using the CA private key:
```
{
  openssl genrsa -out sa.key 2048
  openssl req -new -key sa.key -subj "/CN=service-accounts" -out sa.csr
  openssl x509 -req -in sa.csr -CA ca.crt -CAkey ca.key -CAcreateserial -days 3650 -out sa.crt
}
```
> The following files will be created: `sa.key`, `sa.csr` and `sa.crt`

## Verification
To inspect and verify a specific certificate you can run the following command (in this example I'm inspecting the `kube-apiserver` certificate):
```
openssl x509 -in kube-apiserver.crt -text
```

## Distribute the Client and Server Certificates
Copy the appropriate certificates and private keys to each controller VM:

```
for node in k8s-control-01 k8s-control-02 k8s-control-03; do
  scp ca.key ca.crt kube-apiserver.key kube-apiserver.crt \
      etcd-server.key etcd-server.crt sa.key sa.crt \
      front-proxy-client.key front-proxy-client.crt admin.key admin.crt ${node}:~/
done
```

Copy the CA certificate to each worker VM:
```
for node in k8s-worker-01 k8s-worker-02 k8s-worker-03; do
  scp ca.crt admin.key admin.crt ${node}:~/
done
```

> The `kube-proxy`, `kube-controller-manager`, and `kube-scheduler` client certificates will be used to generate client authentication configuration files in the next lab.

Next: [Generating Kubernetes Configuration Files for Authentication](06-kubernetes-configuration-files.md)
