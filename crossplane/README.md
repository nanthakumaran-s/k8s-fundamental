# Crossplane

To deploy s3 using crossplane, first install the `aws provider` in the k8s cluster

`aws-provider.yml`

```yml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws-s3
spec:
  package: xpkg.upbound.io/upbound/provider-aws-s3:v0.37.0
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

`aws-provider-config.yml`
```yml
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
```

Create a managed resource using crossplane

`s3.yml`

```yml
apiVersion: s3.aws.upbound.io/v1beta1
kind: Bucket
metadata:
  name: $bucket
spec:
  forProvider:
    region: us-east-2
  providerConfigRef:
    name: default
```