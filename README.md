# Crossplane infrastructure as code #

## Overview ##

Compared to the likes of Terraform and Pulumi, [Crossplane](https://www.crossplane.io/) is a relative newcomer to the IaC space. However, as a kubernetes based platform it is appealing by virtue of:

- leveraging our existing k8s skillset
- making use of a large and growing k8s ecosystem

The initial strategy is to create a configuration to deploy the base cluster for EKS then follow that with required services e.g. cert-manager, cluster-autoscaler, external-secrets-operator etc. This means a move to a different CSP will only require the cluster deployment to change.

## Initial Setup ##

The [intial setup instructions](./docs/initial-setup.md) describe how to use and IaC solution that runs in Kubernetes to deploy itself. Thankfully this does not have to be done very often...

# Cluster Deployment #

The cluster deployments use kustomize to install the kubernetes cluster and components.

- Nofrixion specific [composite resource definitions (XRDs) and compositions](../apis/aws/) have beend defined to deploy a VPC and kubernetes cluster to AWS. The configuration is the same as those cluster initially deployed using `eksctl`
- Cluster components (e.g. cluster autoscaler, nginx ingress controller, cert-manager, rabbitmq etc.) are deployed as seperate resources.

This approach improves modularity in terms of deploying clusters that require different components, or in the case of deploying to a different CSP, a different composition for the cluster can be created.

To deploy a cluster, create a kustomization.yaml file to deploy the following resouces to a specific namespace:

* a cluster claim, which calls the xrds and compositions to create a specific cluster instance. For example, the [it-ops-1 cluster](./it-ops-cluster/it-ops-cluster.yaml)
* crossplane objects and releases to deploy additional components. `Objects` use the crossplane kubernetes provider to run the equivalent of `kubectl apply ...` and `Releases` use the helm provider to deploy helm charts.


## Troubleshooting ##

### Deleting 'stuck' resources ###

This mostly happens during testing but, in case of emergency remove `finalisers` from the managed resource e.g.

```bash
TARGET="{YOUR STUCK RESOURCES NAME}"
kubectl patch $TARGET -p '{"metadata":{"finalizers": []}}' --type=merge
```

## REFERENCES ##

* Crossplane docs:
  * [Crossplane composite resource definitions (XRDs)](https://docs.crossplane.io/latest/concepts/composite-resource-definitions/)
  * [Crossplane compositions](https://docs.crossplane.io/latest/concepts/compositions/)
* Anton Putra's [tutorial for creating VPC and deploying EKS](https://youtu.be/mpfqPXfX6mg?si=VK0LR-SfwYGGs6KO) - basically what we want but with only two subnet pairs instead of three.
* [GitOps model for provisioning and bootstrapping Amazon EKS clusters using Crossplane and Argo CD](https://aws.amazon.com/blogs/containers/gitops-model-for-provisioning-and-bootstrapping-amazon-eks-clusters-using-crossplane-and-argo-cd/) - see section, `Amazon EKS cluster provisioning using Crossplane`
* A video showing how to use a temporary local cluster to bootstrap [Crossplane to manage Crossplane](https://youtu.be/IlaYGgyg06o?si=mXM9p73MyrLCd8gA)
* AWS Blueprints (community provider)
  * [Deployment examples](https://github.com/awslabs/crossplane-on-eks/tree/main/examples/aws-provider/composite-resources/eks)
  * [Composition definitions](https://github.com/awslabs/crossplane-on-eks/blob/main/compositions/aws-provider/eks/)
