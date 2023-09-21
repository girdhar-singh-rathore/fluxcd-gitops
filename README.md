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


## source-controller

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

#Reconcilation of Git Source
#Run a flux cmd to manually reconcile a source using below spec:
#Operation: reconcile
#Type: source
#Source: git
#Name: flux-system
flux reconcile source git flux-system

#Reconcilation of Kustomization
#Run a flux cmd to manually reconcile a kustomization using below spec:
#Operation: reconcile
#Type: kustomization
#Name: flux-system
flux reconcile kustomization flux-system

#to access the application
http://localhost:30001/
```


## kustomize-controller

references: https://fluxcd.io/flux/components/kustomize/

```shell
flux get kustomizations

#create source and kustomization manually
#go to repo of bx-game-app
git checkout 2-demo

#create source resource which is going to pull the manifest from any git repo
#with option --export it will give you the yaml file
flux create source git 2-demo-source-git-bx-game-app \
  --url=https://github.com/girdhar-singh-rathore/dx-game-app \
    --branch=2-demo \
    --timeout=10s \
    --export 
    
# create
flux create source git 2-demo-source-git-bx-game-app \
  --url=https://github.com/girdhar-singh-rathore/dx-game-app \
    --branch=2-demo \
    --timeout=10s

#delete the source which was created above using imperative command
flux delete source git 2-demo-source-git-bx-game-app

#store the declarative file yaml specification 
flux create source git 2-demo-source-git-bx-game-app \
  --url=https://github.com/girdhar-singh-rathore/dx-game-app \
    --branch=2-demo \
    --timeout=10s \
    --export > 2-demo-source-git-bx-game-app.yaml

#to deploy above source, we need to create kustomization

# create kustomization
flux create kustomization 2-demo-kustomization-bx-game-app \
  --source=2-demo-source-git-bx-game-app \
  --path="manifests" \
  --prune=true \
  --interval=10s \
  --target-namespace=2-demo \
  --export > 2-demo-kustomization-bx-game-app.yaml

#push the above yaml file to git repo
#apply the kustomization
flux get sources git
flux get kustomizations

#to access the application
http://localhost:30002/
```

### kustomize-controller with kustomize overlay

overlay is a folder which contains the kustomization.yaml file

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yml
- service.yml
- namespace.yml
```

```shell
#create source and kustomization to fetch from 3-demo branch
#go to repo of bx-game-app
#git checkout 3-demo
flux create source git 3-demo-source-git-bx-game-app \
  --url=https://github.com/girdhar-singh-rathore/dx-game-app \
    --branch=3-demo \
    --timeout=10s \
    --export > 3-demo-source-git-bx-game-app.yaml

# create kustomization
flux create kustomization 3-demo-kustomization-bx-game-app \
  --source=3-demo-source-git-bx-game-app \
  --path="manifests" \
  --prune=true \
  --interval=10s \
  --target-namespace=3-demo \
  --export > 3-demo-kustomization-bx-game-app.yaml

flux get source git
flux get kustomizations

http://localhost:30003/
```

### source controller - s3 bucket



```shell
#go to repo of bx-game-app
#git checkout 4-demo
git checkout 4-demo

#we are going to create bucket via 3rd party tool called minio
kubectl apply -f ./minio/*

kubectl get all -n minio-dev
#NAME            TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                         AGE
#service/minio   NodePort   10.105.19.146   <none>        9090:30040/TCP,9000:30041/TCP   75s

#to access the minio
http://localhost:30040/
#credentials are minioadmin/minioadmin

#create bucket on minio portal with name bx-game-app and create path with name app-78
#upload the manifest files to app-78 folder

# create source
flux create source bucket 4-demo-source-minio-s3-bucket-bx-game-app \
    --bucket-name=bx-game-app \
    --secret-ref=minio-secret \
    --endpoint=minio.minio-dev.svc.cluster.local:9000 \
    --provider=generic \
    --insecure=true \
    --export > 4-demo-source-minio-s3-bucket-bx-game-app.yaml

# create kustomization
flux create kustomization 4-demo-kustomization-minio-s3-bucket-bx-game-app \
  --source=Bucket/4-demo-source-minio-s3-bucket-bx-game-app \
  --path="app-78" \
  --prune=true \
  --target-namespace=4-demo \
  --export > 4-demo-kustomization-minio-s3-bucket-bx-game-app.yaml
  

# create secrets for minio with name minio-secret
kubectl create secret -n flux-system generic minio-secret \
    --from-literal=accesskey=minioadmin \
    --from-literal=secretkey=minioadmin

#acess the application
#check the nodeport
kubectl get all -n 4-demo

http://localhost:30275/
```

### helm controller with git as source

```shell
#go to repo of bx-game-app
#git checkout 5-demo
#we have the helm chart in the repo

# create source to fetch the helm chart from git repo
flux create source git 5-demo-source-helm-git-bx-game-app \
  --url=https://github.com/girdhar-singh-rathore/dx-game-app \
  --branch=5-demo \
  --timeout=10s \
  --export > 5-demo-source-helm-git-bx-game-app.yaml

#we can not use the kustomization to deploy the helm chart, we need to use helmrelease

# create helmrelease
flux create helmrelease 5-demo-helmrelease-bx-game-app \
  --source=GitRepository/5-demo-source-helm-git-bx-game-app \
  --chart=./helm-chart \
  --target-namespace=5-demo \
  --values=5-demo-values.yaml \
  --export > 5-demo-helmrelease-bx-game-app.yaml
  
#verify the helmrelease
flux get helmrelease
flux get sources git
kubectl get ns
kubectl get all -n 5-demo
flux get sources all
  
#to access the application
http://localhost:30005/
```


### helm controller with helm repo as source

```shell
# create source to fetch the helm chart from helm repo
flux create source helm 6-demo-source-helm-repo-bx-game-app \
  --url=https://github.com/sidd-harth/block-buster-helm-app \
  --timeout=10s \
  --export > 6-demo-source-helm-repo-bx-game-app.yaml
  
  
# create helmrelease
flux create helmrelease 6-demo-helmrelease-bx-game-app \
  --source=HelmRepository/6-demo-source-helm-repo-bx-game-app \
  --chart=block-buster-helm-app \
  --interval=10s \
  --target-namespace=6-demo \
  --values=6-demo-values.yaml \
  --export > 6-demo-helmrelease-bx-game-app.yaml
  
#verify the helmrelease
flux get helmrelease
flux get sources helm
```


### OCI artifact 

https://opencontainers.org/

### push kubernetes manifest to OCI registry

```shell
#go to repo of bx-game-app
git checkout 7-demo
cd 7.7.0
#login to ghcr.io
docker login ghcr.io -u girdhar-singh-rathore

#create oci artifact and publish it to ghcr.io
flux push artifact oci://ghcr.io/girdhar-singh-rathore/dx-game-app:7.7.0-$(git rev-parse --short HEAD) \
  --path=manifests \
  --source="$(git config --get remote.origin.url)" \
  --revision="7.7.0/$(git rev-parse --short HEAD)"
 
```

### push helm chart to OCI registry

```shell
#go to repo of bx-game-app
git checkout 7-demo
cd 7.7.1
helm package ./helm-chart

#login to ghcr.io helm registry
helm registry login ghcr.io -u girdhar-singh-rathore
#push the helm chart to ghcr.io helm registry
helm push block-buster-helm-app-7.7.1.tgz oci://ghcr.io/girdhar-singh-rathore/dx-game-app

```

### setting up the mysql database

```shell
#go to repo of bx-game-app
git checkout infrastructure
#explore the database folder

#create source countroller for infra repo
flux create source git infra-source-git \
  --url=https://github.com/girdhar-singh-rathore/dx-game-app \
    --branch=infrastructure \
    --timeout=10s \
    --export > infra-source-git.yaml
    
#create kustomization for infra repo, which will only deploy the database folder
flux create kustomization infra-database-kustomization-git-mysql \
  --source=GitRepository/infra-source-git \
  --path="database" \
  --prune=true \
  --interval=10s \
  --target-namespace=database \
  --export > infra-database-kustomization-git-mysql.yaml

#verify the kustomization
flux get kustomizations
flux get sources git
```

### pull and deploy from OCI registry

```shell
# to pull the oci artifact from ghcr.io, you must know the name and tag of the artifact
# create source controller for oci artifact
flux create source oci 7-demo-infra-source-oci-dx-game-app-770 \
  --url=oci://ghcr.io/girdhar-singh-rathore/dx-game-app \
  --tag=7.7.0-14e35a5 \
  --secret-ref=ghcr-secret \
  --provider=generic \
  --export > 7-demo-infra-source-oci-dx-game-app-770.yaml

#create secret for ghcr.io
flux create secret oci ghcr-secret \
  --url=ghcr.io \
  --username=girdhar-singh-rathore \
  --password=<token> 
  
# create kustomization for oci artifact

flux create kustomization 7-demo-infra-kustomization-oci-dx-game-app-770 \
  --source=OciRepository/7-demo-infra-source-oci-dx-game-app-770 \
  --target-namespace=7-demo \
  --interval=10s \
  --prune=false \
  --health-check="Deployment/block-buster-deployment770-demo.7-demo" \
  --depends-on="infra-database-kustomization-git-mysql" \
  --timeout=2m \
  --export > 7-demo-infra-kustomization-oci-dx-game-app-770.yaml

#IMP NOTE: verify the kustomization and check if there is any error
k -n flux-system get kustomizations
# debug specific kustomization
k -n flux-system get kustomizations 7-demo-infra-kustomization-oci-dx-game-app-770 -o yaml

#reconcile the kustomization
flux reconcile kustomization 7-demo-infra-kustomization-oci-dx-game-app-770

flux reconcile source git flux-system

kubectl get all -n 7-demo
```

## image automation controller

https://fluxcd.io/flux/components/image/

install image automation controller

```shell

#check the flux configuraion
kubectl -n flux-system get deploy

#image automation controller is not installed by default, we need to install it
#let use bootstrap command to update the flux configuration, and install the image automation controller

flux bootstrap github \
  --owner=girdhar-singh-rathore \
  --repository=fluxcd-gitops \
  --branch=main \
  --path=clusters/dev \
  --personal=true \
  --private=false \
  --components-extra=image-reflector-controller,image-automation-controller

#verify the image automation controller
kubectl -n flux-system get deploy

#initialize the dockerhub 
#go to repo of bx-game-app
git checkout 8-demo
cd 8.8.0
#login to dockerhub
docker logout 
docker login

#pull the docker image
docker pull siddharth67/block-buster-dev:7.8.0

#tag the docker image
docker tag siddharth67/block-buster-dev:7.8.0 rathore78/dx-game-app:7.8.0

#push the docker image
docker push rathore78/dx-game-app:7.8.0

#replace the image in deployment.yml file of bx-game-app

#create source controller
flux create source git 8-demo-source-git-bx-game-app \
  --url=https://github.com/girdhar-singh-rathore/dx-game-app \
  --branch=8-demo \
  --timeout=10s \
  --export > 8-demo-source-git-bx-game-app.yaml  
  
#create kustomization
flux create kustomization 8-demo-kustomization-git-bx-game-app \
  --source=GitRepository/8-demo-source-git-bx-game-app \
  --prune=true \
  --interval=10s \
  --target-namespace=8-demo \
  --path="manifests" \
  --export > 8-demo-kustomization-git-bx-game-app.yaml


  --target-namespace=8-demo \
  --export > 8-demo-kustomization-git-bx-game-app.yaml
```