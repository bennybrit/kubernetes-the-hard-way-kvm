# Installing the kubectl Client Tool

In this lab, we'll install the [kubectl](https://kubernetes.io/docs/reference/kubectl/overview/) command-line utility which will be used to interact with the Kubernetes API Server.

The `kubectl` tool should be installed on the controller VMs (`k8s-control-01`, `k8s-control-02`, `k8s-control-03`) and worker VMs (`k8s-worker-01`, `k8s-worker-02`, `k8s-worker-03`), so make sure you run the upcoming steps on each one of them.

I'll be using also the KVM host (`k8s-kvm-host`) as the client so I'll install the tool on it too.

## Install kubectl
Download kubectl tool (v1.23.3) and set executable permissions:
```
{
  wget https://dl.k8s.io/release/v1.23.3/bin/linux/amd64/kubectl -O /usr/local/bin/kubectl
  chmod +x /usr/local/bin/kubectl
}
```
Add kubectl autocomplete to the bash shell:
```
kubectl completion bash > /etc/bash_completion.d/kubectl
```

## Verification
Verify kubectl version 1.23.3 is installed:
```
kubectl version --client
```

> Expected output:
```
Client Version: version.Info{Major:"1", Minor:"23", GitVersion:"v1.23.3", GitCommit:"816c97ab8cff8a1c72eccca1026f7820e93e0d25", GitTreeState:"clean", BuildDate:"2022-01-25T21:25:17Z", GoVersion:"go1.17.6", Compiler:"gc", Platform:"linux/amd64"}
```

Next: [Provisioning a CA and Generating TLS Certificates](05-pki-Infrastructure.md)
