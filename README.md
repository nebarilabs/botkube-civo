# botkube-civo
Demonstrate Botkube ChatOps via GitOps

## Pre-Requisite Binaries
You will need the following 3 binaries, even via curl if you're feeling brave and trusting
- helm
- flux
- kustomize
```c
#helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
#flux
curl -s https://fluxcd.io/install.sh | sudo bash
#kustomize
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh"  | bash
```
---
# Installation

## Summary
What we will essentially do is use local helm cli to get the values needed for the chart. Flux will also be setup to say where is the chart to pull from when in cluster and how to define the helmRelease to be reconciled via helm-controller in cluster. Kustomize is used to help validate any changes you would like to make before applying. We will then make a patch by copying the base helmRelease to a patch file that will replace items of interest like a cluster-name or even api tokens

---
## Creating all the things
1. You can use helm to access the repo for the chart
```c
 helm repo add infracloudio https://charts.botkube.io
```

2. Pull down the chart values to be used to make the helmRelease soon
```c
helm show values infracloudio/botkube > values.yaml
```

3. Create a helm repository source to pull the chart from
```c
flux create source helm botkube --url=https://charts.botkube.io --interval=720m --export > helmrepo-botkube.yaml
```

4. Create a helmrelease with flux cli
```c
flux create hr botkube-civo --source=HelmRepository/botkube --chart=botkube --values=./values.yaml --target-namespace=botkube-civo-demo --export > helmrelease-botkube.yaml
``` 

5. Create a kustomize to apply the resources
```c
kustomize create --autodetect --labels botkube-civo:demo,dev:nebarilabs
```

6. You can validate what kustomize will build of all the resources and patches even via kustomize build function
```c
kustomize build . | less
```

7. From here if you want to make a patch for kustomize you can copy the helmrelease as a patch line and add it to the kustomize
```c
cp helmrelease-botkube.yaml patch-clustername.yaml
```
For example you can modify it down to something as simple as setting this one value in the helm chart such as **clusterName*
```yaml
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: botkube
  namespace: flux-system
spec:
  values:
    settings:
      clusterName: botkube-civo-demo

```
8. Append to kustomization a patch for any patch resources like clustername or tokens even, again you can use **kustomize build .** to validate what kustomize will do with the resources and patches
```yaml
patchesStrategicMerge:
- patch-clustername.yaml
```

---
# References
- [Artifacthub.io Botkube](https://artifacthub.io/packages/helm/infracloudio/botkube)
- [Civo Community Slack](https://civo-community.slack.com/archives/CMVCKMCN5)
- [CNCF Slack](https://communityinviter.com/apps/cloud-native/cncf)

## Contacts
- Find me Shawn Garrett on civo community slack and cncf slack
- [Contact Shawn at Nebari Labs](mailto:shawn@nebarilabs.com)
