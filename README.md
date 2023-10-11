# K8s Fundamentals (Crossplane, LitmusChaos, Velero)

## Prerequisites

- Helm
- Velero
- kubectl

## Update kubectl config for AWS Cluster

```bash
aws eks update-kubeconfig --region <REGION> --name <CLUSTER_NAME>
```

## Step 1: Crossplane

### Installation

```bash
helm repo add \
crossplane-stable https://charts.crossplane.io/stable
helm repo update
```

```bash
helm install crossplane \
crossplane-stable/crossplane \
--namespace crossplane-system \
--create-namespace
```

```bash
kubectl get pods -n crossplane-system
```

### Provider and ProviderConfig

```bash
cat <<EOF | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws-s3
spec:
  package: xpkg.upbound.io/upbound/provider-aws-s3:v0.37.0
EOF
```

```bash
kubectl get providers
```

To work with the provider you need to install provider config with the `AWS secret` mentioned

```txt
[default]
aws_access_key_id = <YOUR_KEY>
aws_secret_access_key = <YOUR_KEY>
```

```bash
kubectl create secret \
generic aws-secret \
-n crossplane-system \
--from-file=creds=./aws-credentials.txt
```

```yml
cat <<EOF | kubectl apply -f -
apiVersion: aws.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-secret
      key: creds
EOF
```

```bash
bucket=$(echo "crossplane-bucket-"$(head -n 4096 /dev/urandom | openssl sha1 | tail -c 10))
cat <<EOF | kubectl apply -f -
apiVersion: s3.aws.upbound.io/v1beta1
kind: Bucket
metadata:
  name: $bucket
spec:
  forProvider:
    region: us-east-2
  providerConfigRef:
    name: default
EOF
```

```bash
kubectl get buckets
```

## Step 2: LitmusChaos

```bash
helm repo add litmuschaos https://litmuschaos.github.io/litmus-helm/
helm repo list
```

```bash
kubectl create ns litmus
```

```bash
helm install chaos litmuschaos/litmus --namespace=litmus --set portal.frontend.service.type=LoadBalancer
```

## Step 3: Velero Backup

```bash
velero install \
    --provider aws \
    --plugins velero/velero-plugin-for-aws:v1.0.1 \
    --bucket <BUCKET_NAME> \
    --backup-location-config region=<REGION> \
    --snapshot-location-config region=<REGION> \
    --secret-file ./aws-credentials.txt
```

```bash
velero backup create <BACKUP_NAME> --include-namespaces <NAMESPACE>
```

## Update kubectl config for Azure Cluster

```bash
az aks get-credentials --resource-group <RESOURCE_GROUP> --name <CLUSTER_NAME>
```

## Step 4: Velero Restore

```bash
velero install \
    --provider aws \
    --plugins velero/velero-plugin-for-aws:v1.0.1 \
    --bucket <BUCKET_NAME> \
    --backup-location-config region=<REGION> \
    --snapshot-location-config region=<REGION> \
    --secret-file ./aws-credentials.txt
```

```bash
velero restore create --from-backup <BACKUP_NAME>
```