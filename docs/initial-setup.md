# Installing the Crossoplane management cluster #

## 1. Overview ##

A kubernetes cluster is required to run crossplane. The initial deployent process can be summarised as:

1. deploy local kubernetes cluster
2. install crossplane, dependencies and apis
3. use local cluster to deploy management cluster [manifests](./it-ops-cluster/it-ops-cluster.yaml) to EKS
    - add management cluster details to `~/.kube/config`
    - repeat step 2 above on the management cluster 
    - add the existing vpc id's to the management cluster manifests and redeploy into management cluster

The result should be a management cluster, that is running crossplane and managing itself (as well as other clusters).

## 2. Install local kubernetes cluster ##

This guide describes setting up a local microk8s cluster (in WSL2) to deploy crossplane, but can be adapted to use any local kubernetes flavour e.g. kind, k3s, minikube etc. Microk8s was chosen due to some issues with Rancher desktop messing with the local `.kube/config` file.

It is recommended to create a local cluster soley for crossplane deployment that can be deleted after as stopping the cluster and uninstalling it is the simplest way to 'uncouble' the deployed resources from the temporary cluster (see section 6).

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

## 3. Install and Configure Crossplane ##

### 3.1. Install crossplane system using helm ###

[Install](https://docs.crossplane.io/latest/software/install/) using helm:

```bash
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update

helm install crossplane crossplane-stable/crossplane \
  --set args='{"--enable-usages"}' \
  --namespace crossplane-system --create-namespace

# check it's up and running:
kubectl get pods -n crossplane-system
```

### 3.2. Set up AWS access keys ###

A [crossplane user](https://us-east-1.console.aws.amazon.com/iam/home?region=eu-west-1#/users/details/crossplane-test?section=permissions) has been created in AWS IAM. Create/use an access key and copy access id & secret into [credential file](./aws-credentials.txt) then create a kubernees secret from the file - make SURE this file does NOT get commited to a git repository:

```bash
kubectl create secret generic aws-secret -n crossplane-system --from-file=creds=./aws-credentials.txt
```

For full details see the [AWS Quickstart](https://docs.crossplane.io/latest/getting-started/provider-aws/) has section on setting up a kubernetes secret using an AWS Access Key linked to your account.

### 3.3. Install AWS Providers & NoFrixion apis ###

The necessary providers, configurations and functions can be installed via the [crossplane/dependencies.yaml](./crossplane/dependencies.yaml).

```bash
# Crossplane configurations for aws eks and networks install the necessary providers and functions.
kubectl apply -f crossplane/dependencies.yaml -n crossplane-system
kubectl apply -f crossplane/provider-config-aws.yaml -n crossplane-system

# Check that the providers are all installed and healthy:
kubectl get providers

# Install NoFrixion composite resource definitions and compositions (could package these at some point)
kubectl apply -f apis/aws/vpc/
kubectl apply -f apis/aws/eks-cluster/
```

* Note, some online examples refer to a 'monolithic' `provider-aws` package. This is being deprecated in June 2024. The [`aws provider family`](https://marketplace.upbound.io/providers/upbound/provider-family-aws) packages installed by using the dependencies file above should be used instead. 

## 4. Get KUBECONFIG details from crossplane generated secret ##

```bash
# Set secret name to the appopriate value 
SECRETNAME=$(kubectl -n crossplane-system get secret -o name | grep "-eks-cluster-auth")
kubectl --namespace crossplane-system get $SECRETNAME --output jsonpath="{.data.kubeconfig}" | base64 -d >kubeconfig.txt
```

Once you have the kubeconfig data, merge the Cluster and Context details into an existing `.kube/config` file. Change the user in the `context` to be the azure user used for authenticating with other clusters.

Change context to the management cluster and check connectivity (requires VPN exit node) e.g. `kubectl get nodes` or `kubectl get namespaces`.

## 5. Install crossplane in the management cluster ##

Change context to the management cluster and complete all steps in section 3 on the management cluster.

## 6. Removing the local cluster ##

This step is very imporant. If a managed resource is deleted in crossplane, the real resource will be deleted in the cloud. It is possible to retain the cluster by patching the `deletionPolicy` of all resources to orphan, but it is easier to use a temporary cluster and delete it. E.g.:

```bash
sudo microk8s stop
sudo snap remove microk8s
```