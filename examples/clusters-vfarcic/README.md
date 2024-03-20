# Cloud k8s Clusters #

K8s cluster compositions and custom resource definitions for Amazon, Azure and Google cloud providers created by Viktor Farcic, a developer advocate at Upbound (the people behind Crossplane).

Source: https://github.com/vfarcic/crossplane-kubernetes

## Example ##

The following example creates an EKS cluster in AWS:

```yaml
apiVersion: devopstoolkitseries.com/v1alpha1
kind: ClusterClaim
metadata:
  name: a-team-eks
spec:
  id: a-team-eks
  compositionSelector:
    matchLabels:
      provider: aws
      cluster: eks
  parameters:
    nodeSize: medium
    minNodeCount: 3
```

There are [more examples](https://github.com/vfarcic/crossplane-kubernetes/tree/main/examples) in the github repo.