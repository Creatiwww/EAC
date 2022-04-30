#### EAC
## Setup
```
$ cd initial-crossplane-setup/
```
# Install Kind
```
$ curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.12.0/kind-linux-amd64
$ chmod +x ./kind
$ mv ./kind /usr/local/bin/kind
```
# Run cluster and set kubectl context
```
$ kind create cluster --config kind-config.yaml --name=local-cluster
# check existing contexts: kubectl config get-contexts
$ kubectl config use-context kind-local-cluster
# delete: $ kind delete cluster --name kind-local-cluster
```
# Install Crossplane
```
$ kubectl create namespace crossplane-system
$ helm repo add crossplane-stable https://charts.crossplane.io/stable
$ helm repo update
$ helm install crossplane --namespace crossplane-system crossplane-stable/crossplane

```
# Install Crossplane AWS Provider and set it up
(https://crossplane.io/docs/v1.7/cloud-providers/aws/aws-provider.html)
```
$ kubectl apply -f aws-provider.yaml
$ export AWS_PROFILE=default
$ BASE64ENCODED_AWS_ACCOUNT_CREDS=$(echo -e "[default]\naws_access_key_id = $(aws configure get aws_access_key_id)\naws_secret_access_key = $(aws configure get aws_secret_access_key)" | base64  | tr -d "\n")

$ cat > provider.yaml <<EOF
---
apiVersion: v1
kind: Secret
metadata:
  name: aws-account-creds
  namespace: crossplane-system
type: Opaque
data:
  credentials: ${BASE64ENCODED_AWS_ACCOUNT_CREDS}
---
apiVersion: aws.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-account-creds
      key: credentials
EOF

$ kubectl apply -f "provider.yaml"
$ unset BASE64ENCODED_AWS_ACCOUNT_CREDS
```

## Usage
```
$ cd ../resources
```
# setting up VPC
(https://doc.crds.dev/github.com/crossplane/provider-aws/ec2.aws.crossplane.io/VPC/v1beta1@v0.26.1)
```
$ kubectl apply -f vpc.yaml
$ kubectl apply -f subnet.yaml
$ kubectl apply -f igw.yaml
$ kubectl apply -f rt.yaml

# check: $kubectl get vpc
# delete: $kubectl delete -f ./resources/
```
