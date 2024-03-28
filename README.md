# Botkube-Civo-Demo
Demonstrate Botkube ChatOps via GitOps.

## Botkube Office Hours Presentation
This is the video that implements and demonstrates this repo creating a new cluster and deploying with GitOps practices to deploy Botkube to achieve ChatOps.
[![Botkube Office Hours Presentation](https://www.youtube.com/live/WxkDWLqVghw?si=d8cQudgtyUF-KdPT)]

## Prerequisites
You will need the following 5 binaries, even via curl if you're feeling brave and trusting, its assumed you have kubectl & civo cli's already.
- helm
- flux
- kustomize
- civo
- kubectl
```c
#helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
#flux
curl -s https://fluxcd.io/install.sh | bash
#kustomize
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh"  | bash
#civo
curl -sL https://civo.com/get | sh
#kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```
---
# Installation

## Summary
What we will essentially do is use local [helm](https://helm.sh/) cli to get the values needed for the [Botkube](https://botkube.io/) helm chart. [Flux](https://fluxcd.io/) will also be setup with the chart's repository as a source so the in cluster operator can have the defined helmRelease to reconciled via the [helm-controller](https://fluxcd.io/flux/components/helm/). [Kustomize](https://kustomize.io/) is used layer on some minor changes and customizatins as well to help validate any changes we would like to make before applying. We will then make a patch by copying the base helmRelease to a patch file that will replace items of interest like a cluster-name, available executors, api tokens, etc. After quickly creating a cluster in our favorite kubernetes cloud provider [Civo](https://civo.com/) we give it our manifest to watch the GitOps reconcile our desired state and manage the helm release for us.

### Architecture Overview
![high level overview](https://github.com/nebarilabs/botkube-civo-demo/blob/main/civo-flux-helmController-botkube-discord.png?raw=true)

---
## Creating all the things
1. You can use helm to access the repo for the chart
```c
 helm repo add infracloudio https://charts.botkube.io
```

2. Update your local computer's helm repo cache so that we'll be able to pull down the charts local **values.yaml** as input for the next step
```c
helm repo update
```

3. Pull down the chart values to be used to make the helmRelease soon
```c
helm show values infracloudio/botkube > values.yaml
```

4. Create a helm repository source to pull the chart from
```c
flux create source helm botkube --url=https://charts.botkube.io --interval=720m --export > helmrepo-botkube.yaml
```

5. Create a helmrelease with flux cli
```c
flux create hr botkube-civo-demo --source=HelmRepository/botkube --chart=botkube --values=./values.yaml --target-namespace=botkube-civo-demo --export > helmrelease-botkube.yaml
``` 

6. Create the namespace up front
```c
kubectl create ns botkube-civo-demo --dry-run=client -o yaml > namespace-botkube-civo-demo.yaml
```

7. Create a kustomize to apply the resources
```c
kustomize create --autodetect --labels botkube-civo:demo,dev:nebarilabs --namespace botkube-civo-demo
```

8. You can validate what kustomize will build of all the resources and patches even via kustomize build function
```c
kustomize build . | less
```

9. From here if you want to make a patch for kustomize you can copy the helmrelease as a patch line and add it to the kustomize
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
10. Append to kustomization a patch for any patch resources like clustername or tokens even, again you can use **kustomize build .** to validate what kustomize will do with the resources and patches
```yaml
patchesStrategicMerge:
- patch-clustername.yaml
```

11. Create a cluster in civo cloud to deploy to in this example its called **test04**
```c
date && time civo k8s create test04 --region NYC1 --save --merge --nodes 1 --size g4s.kube.small --cluster-type k3s --switch --wait -a Traefik-v2-loadbalancer -a civo-cluster-autoscaler --version 1.27.1-k3s1
```

12. Flux bootstrap the cluster to your repo of choice with the helm-controller
```c
flux bootstrap github   --owner=$GITHUB_USER   --repository=someCoolRepository   --branch=main   --path=datacenters/civo/clusters/test04   --personal --verbose --components=kustomize-controller,source-controller,notification-controller,helm-controller --interval=1m
```

13. With our new cluster available with flux we pass our manifests we've made to the cluster's helm-controller
```c
kubectl apply -k .
```

---
# References
- [Artifacthub.io Botkube](https://artifacthub.io/packages/helm/infracloudio/botkube)
- [Botkube Discord Instructions](https://docs.botkube.io/installation/discord/self-hosted)
- [Civo Community Slack](https://civo-community.slack.com/archives/CMVCKMCN5)
- [CNCF Slack](https://communityinviter.com/apps/cloud-native/cncf)
- [Botkube Office Hours](https://www.youtube.com/live/WxkDWLqVghw?si=d8cQudgtyUF-KdPT)

## Contacts
- Find me Shawn Garrett on the Civo community slack and cncf slack
- [Contact Shawn at Nebari Labs](mailto:shawn@nebarilabs.com)
- [Civo Ambassadors](https://www.civo.com/ambassadors)
