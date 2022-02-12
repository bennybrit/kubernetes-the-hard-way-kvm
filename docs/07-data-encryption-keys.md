# Generating the Data Encryption Config and Key

Kubernetes stores a variety of data including cluster state, application configurations, and secrets. Kubernetes supports the ability to [encrypt](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data) cluster data at rest.

In this lab, we'll generate an encryption key and an [encryption config](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#understanding-the-encryption-at-rest-configuration) suitable for encrypting Kubernetes Secrets.

We'll generate it on the KVM host (`k8s-kvm-host`), and from it, we'll distribute the configuration file to the different Kubernetes controller VMs.

## The Encryption Key
Generate an encryption key:
```
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
```

## The Encryption Config File
Create the `encryption-config.yaml` encryption config file:

```
cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF
```

## Distribute the Encryption Config File
Copy the `encryption-config.yaml` encryption config file to each controller VM:

```
for node in k8s-control-01 k8s-control-02 k8s-control-03; do
  scp encryption-config.yaml ${node}:~/
done
```

Next: [Bootstrapping the etcd Cluster](08-bootstrapping-etcd.md)
