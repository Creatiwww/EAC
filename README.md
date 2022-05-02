# AWS infra provisioning with Crossplane and Helm

## Usage
### Setting up VPC via Helm
(https://doc.crds.dev/github.com/crossplane/provider-aws/ec2.aws.crossplane.io/VPC/v1beta1@v0.26.1)
```
$ cd ../charts
$ helm install playground_vpc --generate-name

 # install with custom values: $ helm install -f <my_values.yaml> playground_vpc --generate-name
 # delete: $ helm uninstall <generated-release-name>
```

## Setup

### Install Kind
```
$ curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.12.0/kind-linux-amd64
$ chmod +x ./kind
$ mv ./kind /usr/local/bin/kind
```

### Make LB working with Kind
(https://kind.sigs.k8s.io/docs/user/loadbalancer/)
```
$ kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.12.1/manifests/namespace.yaml
$ kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.12.1/manifests/metallb.yaml
$ kubectl apply -f https://kind.sigs.k8s.io/examples/loadbalancer/metallb-configmap.yaml
```

### Run cluster and set kubectl context
```
$ kind create cluster --config initial-setup/kind-config.yaml --name=local-cluster

# check existing contexts: $ kubectl config get-contexts
# switch to context: $ kubectl config use-context kind-local-cluster
# delete: $ kind delete cluster --name local-cluster
```

### Install Crossplane
```
$ kubectl create namespace crossplane-system
$ helm repo add crossplane-stable https://charts.crossplane.io/stable
$ helm repo update
$ helm install crossplane --namespace crossplane-system crossplane-stable/crossplane

```
### Install Crossplane AWS Provider and set it up
(https://crossplane.io/docs/v1.7/cloud-providers/aws/aws-provider.html)
```
$ kubectl apply -f initial-setup/aws-provider.yaml
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

### Install Argo CD
(https://argo-cd.readthedocs.io/en/stable/getting_started/)
 ```
# server
$ kubectl create namespace argocd
$ kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
$ kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

# cli
$ curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
$ chmod +x /usr/local/bin/argocd

# login
$ export ARGOCD_SERVER=`kubectl get svc argocd-server -n argocd -o json | jq --raw-output '.status.loadBalancer.ingress[0].ip'`
$ export ARGO_PWD=`kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d`

$ argocd login $ARGOCD_SERVER --username admin --password $ARGO_PWD --insecure
# or just make port forwarding, then: $ argocd login localhost:34675 --username admin --password $ARGO_
```

### Argo CD app setup
 ```
argocd app create helloworld \
--repo https://github.com/ironcore864/go-hello-http.git \
--path helm \
--sync-policy automatic \
--dest-server https://kubernetes.default.svc \
--dest-namespace default \
--values values.yaml
```
