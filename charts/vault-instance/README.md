# How to initialize vault ha raft cluster with cilium
- install helm `vault_operator` from https://kubernetes-charts.banzaicloud.com
- install helm `vault-instance` chart from https://github.com/lop3z/helm-charts

# Variables

## PRIMARY
`cluster_primary_region`: sgp1

`cluster_primary_region_nb_instance`: 2
## SECONDARY
`cluster_secondary_region_1`
`cluster_secondary_region_1_nb_instance`: 1
`cluster_secondary_region_2`
`cluster_secondary_region_2_nb_instance`: 2

# Installation
## Primary instance
1. install the __primary vault__, set the helm release name to 
`vault-${cluster_primary_region}` it will be installed into the region SGP1 with this helm values: 

```yaml
- chart: lop3z/vault-instance
  name: vault-${cluster_primary_region}
  values:
  - primary: true
  - multiRegion:
    - ${cluster_primary_region}
    - ${cluster_secondary_region_1}
    - ${cluster_secondary_region_2}
  - size: ${cluster_primary_region_nb_instance}
  - image: vault:1.6.2
  - tlsAdditionalHosts:
    - vault-${cluster_primary_region}
    - vault-${cluster_primary_region}-0
    - vault-${cluster_primary_region}-1
    - vault-${cluster_secondary_region_1}
    - vault-${cluster_secondary_region_1}-0
    - vault-${cluster_secondary_region_2}
    - vault-${cluster_secondary_region_2}-0
    - vault-${cluster_secondary_region_2}-1
```

2. on kubernetes, check the namespace `jx-vault` and get the secret `vault-${cluster_primary_region}-tls`, this secret must be copy to all cluster, to allow raft cluster authentication.

```
kubectl -n jx-vault get secrets vault-sgp1-tls -o json | jq 'del(.metadata.ownerReferences)' | jq 'del(.metadata.resourceVersion)' | jq 'del(.metadata.uid)' | jq '.metadata.name = "vault-tls"' > vault-tls.json
```` 

3. Copy the secret `vault-unseal-keys` this secret must be copy to all cluster, to be able to unseal the vault

## Secondary instance





- unseal secret will be generate automatically, you will find unseal keys and root token in the secret name `vault-primary-unseal-keys`, this secret must be copied to all cluster

```
kubectl -n jx-vault get secrets vault-unseal-keys -o json | jq 'del(.metadata.ownerReferences)' | jq 'del(.metadata.resourceVersion)' | jq 'del(.metadata.uid)' > vault-unseal-keys.json
````

## Secondary instances

1. Install secondary vault instance with these helm values 

```yaml

chart: lop3z/vault-instance
  name: vault-${cluster_primary_region}
  values:
  - primary: false
  - raftLeaderAddress: "vault-${cluster_primary_region}.jx-vault.svc.cluster.local"
  - multiRegion:
    - ${cluster_primary_region}
    - ${cluster_secondary_region_1}
    - ${cluster_secondary_region_2}
  - size: ${cluster_secondary_region_1_nb_instance}
  - image: vault:1.6.2
  - tlsAdditionalHosts:
    - vault-${cluster_primary_region}
    - vault-${cluster_primary_region}-0
    - vault-${cluster_primary_region}-1
    - vault-${cluster_secondary_region_1}
    - vault-${cluster_secondary_region_1}-0
    - vault-${cluster_secondary_region_2}
    - vault-${cluster_secondary_region_2}-0
    - vault-${cluster_secondary_region_2}-1
```

2. Copy the secret `vault-unseal-keys` and `vault-${cluster_primary_region}-tls` to the secondary instances

`kubectl -n jx-vault apply -f vault-tls.json`

`kubectl -n jx-vault apply -f vault-unseal-keys.json`


# VAULT

1. for each cluster, create a service account token
```
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: vault-k8s-auth-secret
  annotations:
    kubernetes.io/service-account.name: vault
type: kubernetes.io/service-account-token
EOF
```

## Vault authentification methods

1. each cluster must have a kubernetes auth method eg: (sgb5, sgp1)
![Alt text](<Screenshot from 2023-06-21 18-11-28.png>)

2. Configure each authentification like this:
![Alt text](image.png)


__kubernetes_host:__ `get the host from your ~/.kube/config`

__kubernetes_ca:__ `get the host from your ~/.kube/config`

__disable_jwt_issuer_validation:__ `true`

__jwt_issuer:__ `vault-secrets-webhook`

__token_reviewer_jwt :__ `kubectl -n jx-vault get secret vault-k8s-auth-secret -o json | jq -r '.data.token' | base64 -d`


## Create the vault policy
![Alt text](image-1.png)

## Create roles __for each__ vault authentification method
![Alt text](image-3.png)

![Alt text](image-4.png)
# Troubleshooting