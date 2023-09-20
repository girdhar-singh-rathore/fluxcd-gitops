# Get Started with Flux

references: https://fluxcd.io/flux/get-started/

### Install the Flux CLI
https://fluxcd.io/flux/installation/

```bash
brew install fluxcd/tap/flux
```

## Flux bootstrap

references: https://fluxcd.io/flux/installation/bootstrap/

we are using github as a git repository
https://fluxcd.io/flux/installation/bootstrap/github/

```bash
flux bootstrap github \
  --owner=girdhar-singh-rathore \
  --repository=fluxcd-gitops \
  --branch=main \
  --path=clusters/dev \
  --personal=true \
  --private=false
```

```shell
kubectl get all -n flux-system
kubectl get secret flux-system -n flux-system -o yaml
kubectl get crds | grep flux
flux version
```


### source-controller

```shell
flux get sources
flux get sources git
flux get kustomization
```

#### check the flux source-controller package

```shell
kubectl get po -n flux-system

#find the correct pod name
kubectl exec -it -n flux-system source-controller-594c848975-dw2xt -- sh

cd data/gitrepository/flux-system/flux-system/

ls -lrt

#extract the package and see the all deployment and service yaml
tar -tf 6ddf7efbddde4ca44b7088c932c81d3da5324cec.tar.gz

kubectl get all -n 1-demo

#check the nodeport and access the application
kubectl get svc -n 1-demo
```

#### manual reconciliation

```shell
#Check deployment replicas, As of now the app deployment has only 1 replica, which results in 1 running pod.
kubectl -n 1-demo get pod

#Increase Replicas and Commit, Edit deployment.yml and increase the replicas from 1 to 3 and save the file

```

