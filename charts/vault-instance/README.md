# How to initialize vault ha raft cluster with cilium

- install helm `vault-instance` chart from https://github.com/lop3z/helm-charts

## Primary instance
1. install the __primary vault__, helm release name must be `vault-primary` it will be installed into the region SGP1 with this helm values: 

```
primary: true
tlsAdditionalHosts:
 - vault-primary
 - vault-primary-0
 - vault-secondary
 - vault-secondary-0
 - ...
```

- certificate will be generate automatically and you will find it in the secret name `vault-primary-tls`, this certificate must be copied to all cluster

```
kubectl -n jx-vault get secrets vault-primary-tls -o json | jq 'del(.metadata.ownerReferences)' | jq 'del(.metadata.resourceVersion)' | jq 'del(.metadata.uid)' > vault-primary-tls.json
````

- unseal secret will be generate automatically, you will find unseal keys and root token in the secret name `vault-primary-unseal-keys`, this secret must be copied to all cluster

```
kubectl -n jx-vault get secrets vault-primary-unseal-keys -o json | jq 'del(.metadata.ownerReferences)' | jq 'del(.metadata.resourceVersion)' | jq 'del(.metadata.uid)' > vault-primary-unseal-keys.json
````

## Secondary instances

this helm values: 

```
primary: false
raftLeaderAddress: "vault-primary.jx-vault.svc.cluster.local"
tlsAdditionalHosts:
 - vault-primary
 - vault-primary-0
 - vault-secondary
 - vault-secondary-0
 - ...
```