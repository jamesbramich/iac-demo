# Crossplane infrastructure as code #

## Overview ##

Compared to the likes of Terraform and Pulumi, [Crossplane](https://www.crossplane.io/) is a relative newcomer to the IaC space. However, as a kubernetes based platform it is appealing by virtue of:

- leveraging our existing k8s skillset
- making use of a large and growing k8s ecosystem

The initial strategy is to create a configuration to deploy the base cluster for EKS then follow that with required services e.g. cert-manager, cluster-autoscaler, external-secrets-operator etc. This means a move to a different CSP will only require the cluster deployment to change.

## Initial setup ##

A kubernetes cluster is required to run crossplane. The initial deployent process can be summarised as:

1. deploy local kubernetes cluster
2. install crossplane, dependencies and apis
3. deploy [management cluster manifests](./it-ops-cluster/it-ops-cluster.yaml) to EKS
    - add management cluster details to `~/.kube/config`
    - repeat step 2 above on the management cluster 
    - add the existing vpc id's to the management cluster manifests and deploy into management cluster

### Install local kubernetes cluster ###

This pilot uses a local microk8s cluster, but can be applied using any local kubernetes flavour e.g. kind, k3s, minikube etc. I had issues with Rancher desktop messing with my existing confg so went with microk8s in WSL2.

See [MicroK8s docs](https://microk8s.io/docs/install-wsl2) for details. 

TLDR:

```bash
cat /etc/wsl.conf` 
# check [boot] section fro `systemd=true` if not:
# echo -e "[boot]\nsystemd=true" | sudo tee /etc/wsl.conf

#install
sudo snap install microk8s --classic
sudo usermod -a -G microk8s $USER
sudo microk8s enable storage
sudo microk8s enable rbac

# if you want to add the microk8s config to your existing kubectl run `sudo microk8s config`
# and copy the cluster, context and user to your .kube/config file.
```

### Install Crossplane ###

[Install](https://docs.crossplane.io/latest/software/install/) using helm:

```bash
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update

helm install crossplane crossplane-stable/crossplane --namespace crossplane-system --create-namespace 

# check it's up and running:
kubectl get pods -n crossplane-system
```

### Set up AWS access keys ###

Create an AWS Test user/Access key and copy access id & secret into [credential file](./aws-credentials.txt) then create a kubernees secret from the file:

```bash
kubectl create secret generic aws-secret -n crossplane-system --from-file=creds=./aws-credentials.txt
```

For full details see the [AWS Quickstart](https://docs.crossplane.io/latest/getting-started/provider-aws/) has section on setting up a kubernetes secret using an AWS Access Key linked to your account.

### Install AWS Providers & NoFrixion apis###

The necessary providers, configurations and functions can be installed via the [crossplane/dependencies.yaml](./crossplane/dependencies.yaml).

```bash
# Crossplane configurations for aws eks and networks install the necessary providers and functions.
kubectl apply -f crossplane/dependencies.yaml -n crossplane-system
kubectl apply -f crossplane/provider-config-aws.yaml -n crossplane-system

# Install NoFrixion composite resource definitions and compositions (could package these at some point)
kubectl apply -f apis/aws/vpc/
kubectl apply -f apis/aws/eks-cluster/
```

* Note, some online examples refer to a 'monolithic' `provider-aws` package. This is being deprecated in June 2024. The [`aws provider family`](https://marketplace.upbound.io/providers/upbound/provider-family-aws) packages installed by using the dependencies file above should be used instead. 

### Get KUBECONFIG details from crossplane generated secret ###

```bash
# Set secret name to the appopriate value 
SECRETNAME=$(kubectl -n crossplane-system get secret -o name | grep ekscluster)
kubectl --namespace crossplane-system get $SECRETNAME --output jsonpath="{.data.kubeconfig}" | base64 -d >kubeconfig.txt
```

Once you have the kubeconfig data, merge the Cluster and Context details into an existing `.kube/config` file. Change the user in the `context` to be the azure user used for authenticating with other clusters.

## Deleting 'stuck' resources ##

This mostly happens during testing but, in case of emergency remove `finalisers` from the managed resource e.g.

`kubectl patch cluster ops-pilot-cluster-wpcsc-ggspf -p '{"metadata":{"finalizers": []}}' --type=merge`

## REFERENCES ##

* Anton Putra's [tutorial for creating VPC and deploying EKS](https://youtu.be/mpfqPXfX6mg?si=VK0LR-SfwYGGs6KO) - basically what we want but with only two subnet pairs instead of three.
* [GitOps model for provisioning and bootstrapping Amazon EKS clusters using Crossplane and Argo CD](https://aws.amazon.com/blogs/containers/gitops-model-for-provisioning-and-bootstrapping-amazon-eks-clusters-using-crossplane-and-argo-cd/) - see section, `Amazon EKS cluster provisioning using Crossplane`
* A video showing how to use a temporary local cluster to bootstrap [Crossplane to manage Crossplane](https://youtu.be/IlaYGgyg06o?si=mXM9p73MyrLCd8gA)
* AWS Blueprints (community provider)
  * [Deployment examples](https://github.com/awslabs/crossplane-on-eks/tree/main/examples/aws-provider/composite-resources/eks)
  * [Composition definitions](https://github.com/awslabs/crossplane-on-eks/blob/main/compositions/aws-provider/eks/)
