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


## source-controller and kustomize-controller

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


### kustomize-controller

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
    --timeout=1m \
    --export 
    
# create
flux create source git 2-demo-source-git-bx-game-app \
  --url=https://github.com/girdhar-singh-rathore/dx-game-app \
    --branch=2-demo \
    --timeout=1m

#delete the source which was created above using imperative command
flux delete source git 2-demo-source-git-bx-game-app

#store the declarative file yaml specification 
flux create source git 2-demo-source-git-bx-game-app \
  --url=https://github.com/girdhar-singh-rathore/dx-game-app \
    --branch=2-demo \
    --timeout=1m \
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

## helm controller and OCI registry

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
  --export > infra-database-kustomization-git-mysql.yaml_bkp

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

#verify the kustomization
flux get kustomizations
flux get sources git
kubectl -n 8-demo get all

```

### image automation controller 

```shell
flux create image repository 8-demo-image-repository-bx-game-app \
  --image=docker.io/rathore78/dx-game-app \
  --interval=10s \
  --export > 8-demo-image-repository-bx-game-app.yaml

#reconcile the image repository
flux reconcile source git flux-system
#verify the image repository
flux get image repository

#to use automation controller to pick the latest image, we need to create new image and tag it with latest
# go to repo of bx-game-app
# git checkout 8-demo, modify the background color
# don't push the changes, we will create the new image and tag it with latest
docker build -t rathore78/dx-game-app:7.8.1 .

#push the new image
docker push rathore78/dx-game-app:7.8.1

#reconcile the image repository will automatically pick the latest image
flux get image repository
# we can see it found two tags 
# 7.8.0 and 7.8.1
k -n flux-system get imagerepositories 
k -n flux-system get imagerepositories 8-demo-image-repository-bx-game-app -o yaml

#  lastScanResult:
#    latestTags:
#    - 7.8.1
#    - 7.8.0

#image automation controller with image policy will resolve the image with policy

#create image policy
flux create image policy 8-demo-image-policy-bx-game-app \
  --image-ref=8-demo-image-repository-bx-game-app \
  --select-semver="7.8.x" \
  --export > 8-demo-image-policy-bx-game-app.yaml
  
#verify the image policy
flux get image all

[//]: # (fluxcd-gitops git:&#40;main&#41; flux get image all)

[//]: # (NAME                                                    LAST SCAN                       SUSPENDED       READY   MESSAGE                       )

[//]: # (imagerepository/8-demo-image-repository-bx-game-app     2023-09-22T08:53:36+02:00       False           True    successful scan: found 2 tags   )

[//]: # ()
[//]: # (NAME                                            LATEST IMAGE                            READY   MESSAGE                                                                  )

[//]: # (imagepolicy/8-demo-image-policy-bx-game-app     docker.io/rathore78/dx-game-app:7.8.1   True    Latest image tag for 'docker.io/rathore78/dx-game-app' resolved to 7.8.1  )

```

### image automation controller update

it will update the image in deployment

```shell
# create image update 
flux create image update 8-demo-image-update-bx-game-app \
  --git-repo-ref=8-demo-source-git-bx-game-app \
  --checkout-branch=8-demo \
  --author-name=fluxcdbot \
  --author-email=rathore.girdharsingh@gmail.com \
  --git-repo-path=manifests \
  --push-branch=8-demo \
  --interval=10s \
  --export > 8-demo-image-update-automation-bx-game-app.yaml
  
#reconcile 
flux reconcile source git flux-system

#verify the image update
flux get image all 

#NAME                                                    LAST RUN                        SUSPENDED       READY   MESSAGE         
#imageupdateautomation/8-demo-image-update-bx-game-app   2023-09-22T09:24:52+02:00       False           True    no updates made 

# no updates made 
# because in manifest we need to mention which deployment we want to update
#add the marker in deployment.yml file
# image: rathore78/dx-game-app:7.8.0 # {"$imagepolicy": "flux-system:8-demo-image-policy-bx-game-app"}

# now it will fail for authentication issue

#flux does not have permissions to push the dx-game-app git repo

#go you the git repo and add deploy key to the repo, settings -> deploy keys -> add deploy key

#first create key from flux cli
flux create secret git 8-demo-git-dx-game-app-auth \
  --url=ssh://git@github.com/girdhar-singh-rathore/dx-game-app.git \
  --ssh-key-algorithm=ecdsa \
  --ssh-ecdsa-curve=p521 
  
#update the git source with above secret

flux create source git 8-demo-source-git-bx-game-app \
  --url=ssh://git@github.com/girdhar-singh-rathore/dx-game-app.git \
  --branch=8-demo \
  --timeout=10s \
  --secret-ref=8-demo-git-dx-game-app-auth \
  --export > 8-demo-source-git-bx-game-app.yaml 
  

#reconcile the git source
flux reconcile source git flux-system

  
```

## secret management and sign verifications

### bitnami sealed secrets

```shell
#create kustomization for sealed secrets
flux create kustomization infra-security-kustomize-git-sealed-secrets \
  --source=GitRepository/infra-source-git \
  --prune=true \
  --interval=1h \
  --path=bitnami-sealed-secrets \
  --export > infra-security-kustomize-git-sealed-secrets.yaml
  
 
#reconcile the source
flux reconcile source git flux-system

#verify the sealed secrets
k -n kube-system get all
k -n kube-system get secret -l sealedsecrets.bitnami.com/sealed-secrets-key

#encrypt the secret using kubeseal cli
#install kubeseal cli
brew install kubeseal
kubeseal --version

#in order to encrypt the secret, we need to have the public key certificate

#fetch the public key certificate from sealed secret 
k -n kube-system get secret sealed-secrets-keyhfnzz -o yaml

# or we can use the kubeseal cli to fetch the public key certificate
kubeseal --fetch-cert \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system \
    > sealed-secrets.pub

```

### encrypt and decrypt the secret using bitnami sealed secrets

```shell
k -n database get po,secrets
k -n database delete secrets secret-mysql

#but reconcile the kustomization will recreate the secret

#in order to delete fisrt suspend the reconciliation
flux suspend kustomization infra-database-kustomization-git-mysql

#check the status of suspended reconciliation
flux get kustomizations

#now delete the secret
k -n database delete secrets secret-mysql

#restart the deployment
k -n database rollout restart deployment/mysql

#check the status of deployment, it will be in pending state because secret is not created
k -n database get po

#go to repo of bx-game-app
#now encrypt the secret using kubeseal cli
kubeseal -o yaml --scope cluster-wide --cert ../../sealed-secrets.pub < secret-mysql.yaml > sealed-secret-mysql.yamlkubeseal -o yaml --scope cluster-wide --cert ../sealed-secrets.pub < secret-mysql.yaml > sealed-secret-mysql.yaml

#reconcile the source
flux reconcile source git flux-system

#resume the kustomization reconciliation
flux resume kustomization infra-database-kustomization-git-mysql

#verify the secret is created
k -n database get secrets

#get secret 
k -n database get secret secret-mysql -o json | jq -r .data.password -r
#decode the secret
k -n database get secret secret-mysql -o json | jq -r .data.password -r | base64 -d
```


## mozilla sops - screts opertaions with PGP (pretty good privacy), GPG (GNU privacy guard) ADMIN
PGP and GPG are data encryption and decryption program, used for transfering data securely

```shell

#generate the pgp key or gpg key with no passphrase
berw install gpg

gpg --batch --full-generate-key <<EOF
%no-protection
Key-Type: 1
Key-Length: 3072
Subkey-Type: 1
Subkey-Length: 3072
Expire-Date: 0
Name-Real: dev.us.e1.k8s
Name-Email: admin@bb.com
EOF

#access the public key
gpg --list-public-keys 

#access the private key
gpg --list-secret-keys

#export the private key and store in git repo, public key can be used to encrypt the secret
gpg --export-secret-keys --armor 

gpg --export-secret-keys --armor > sops-gpg.key

#export public key
gpg --export --armor > sops-gpg.pub

#create kubernetes secret with private key
kubectl -n flux-system create secret generic sops-gpg \
  --from-file=sops-gpg.key
  
# delete all keys from gpg
gpg --delete-secret-and-public-keys 

```

### mozilla sops - screts opertaions with PGP (pretty good privacy), GPG (GNU privacy guard) DEVELOPER

```shell
#go to repo of bx-game-app
# import the public key
gpg --import ./sops/sops-gpg.pub

#install the sops
brew install sops
sops -v

#encrypt the secret using sops
sops --encrypt \
--encrypted-regex '^(data|stringData)$' \
--pgp 7F8E4700298D54FC \
--in-place secret-mysql.yaml

#reconcile the source
flux reconcile source git flux-system
#recocile the kustomization
flux reconcile kustomization infra-database-kustomization-git-mysql

```


## cosign verification

cosign: A tool for Container Signing, Verification and Storage in an OCI registry.


```shell
brew install cosign
cosign version

#generate key pair
cosign generate-key-pair

#create secrets for cosign to store the public key
kubectl -n flux-system create secret generic cosign-pub \
  --from-file=cosign.pub
 
```


### sign and verify the oci artifact using cosign

```shell

#go to repo of bx-game-app
git checkout 10-demo

#create the oci artifact and publish it to ghcr.io

#login to ghcr.io
docker login ghcr.io -u girdhar-singh-rathore


flux push artifact oci://ghcr.io/girdhar-singh-rathore/dx-game-app:7.10.0-$(git rev-parse --short HEAD) \
  --path=manifests \
  --source="$(git config --get remote.origin.url)" \
  --revision="7.10.0/$(git rev-parse --short HEAD)"
  
 # go to github repo and check the artifact is published 
 
 #sign the artifact using cosign, use the sha256 instead of tag
 
cosign sign -key cosign.key ghcr.io/girdhar-singh-rathore/dx-game-app@sha256:031db7d26174edba990eda99e11843592905be5234e96386bf4e0603c2343d5b

# it will ask for the password for the private key which you have created earlier

#verify the signature of the artifact on ghcr.io registry, it will show the signature .sig file
#if you open the signature file, it will show you signed by the cosign

#manullly verify the artifact using cosign public key
cosign verify -key cosign.pub ghcr.io/girdhar-singh-rathore/dx-game-app@sha256:031db7d26174edba990eda99e11843592905be5234e96386bf4e0603c2343d5b

```

### verify the artifact while fetching from the oci repo, using cosign with image policy

will use the flux oci component to fetch the artifact from ghcr.io

```shell

#create source to fetch artifact from ghcr.io oci
flux create source oci 10-demo-source-oci-dx-game-app \
  --url=oci://ghcr.io/girdhar-singh-rathore/dx-game-app \
  --tag=7.10.0-f0f5090 \
  --secret-ref=ghcr-secret \
  --provider=generic \
  --export > 10-demo-source-oci-dx-game-app.yaml

#in order to verify the artifact, we need to add verify element in exported yaml file
#verify:
#  provider: cosign
#  secretRef:
#    name: cosign-pub

#add kustomization to deploy the artifact
flux create kustomization 10-demo-kustomization-oci-dx-game-app \
  --source=OciRepository/10-demo-source-oci-dx-game-app \
  --target-namespace=10-demo \
  --interval=10s \
  --prune=false \
  --export > 10-demo-kustomization-oci-dx-game-app.yaml

# reconcile the source
flux reconcile source git flux-system
# reconcile the kustomization
flux reconcile kustomization 10-demo-kustomization-oci-dx-game-app
```

## notification controller

```shell
#webhook receiver
#go to repo of bx-game-app
git checkout 2-demo

# git to github repo and create webhook receiver
# bx-game-app -> settings -> webhooks -> add webhook

#expose notification controller
kubectl -n flux-system expose deployment notification-controller \
  --name=receiver \
  --port=80 \
  --target-port=9292 \
  --type=NodePort \

kubectl -n flux-system get svc

#create secret for github webhook
kubectl -n flux-system create secret generic github-webhook-secret \
  --from-literal=token=secret-token-dont-share 

#we need to put "secret-token-dont-share" secrets text in github webhook secret

#create notification receiver object
flux create receiver github-webhook-receiver \
    --type=github \
    --event=ping,push \
    --secret-ref=github-webhook-secret \
    --resource=GitRepository/2-demo-source-git-bx-game-app \
    --export > github-webhook-receiver.yaml
    
#reconcile the source
flux reconcile source git flux-system
#reconcile the receiver
flux reconcile receiver github-webhook-receiver
#reconcile the kustomization
flux reconcile kustomization 2-demo-kustomization-bx-game-app
#verify the receiver
flux get receiver github-webhook-receiver

#NOTE: watch the source
flux get sources git 2-demo-source-git-bx-game-app --watch

#expose the nodeport to local tunnel so that github can send the webhook to notification controller via internet
brew install localtunnel
lt --port 3000 --subdomain girdhar-singh-rathore

k get svc receiver -n flux-system
#receiver   NodePort   10.109.13.202   <none>        80:32318/TCP   127m

flux get receivers
#NAME                    SUSPENDED       READY   MESSAGE                                                                                               
#github-webhook-receiver False           True    Receiver initialized for path: /hook/ab09b3ac5436ef9fca17e74ddc7d57b24be7943a1e00ee34f45df11b3b73beda 
lt --port 32318
#your url is: https://early-coats-chew.loca.lt/hook/ab09b3ac5436ef9fca17e74ddc7d57b24be7943a1e00ee34f45df11b3b73beda  

#get the path from receiver and url from localtunnel
# git to github repo and create webhook receiver
# bx-game-app -> settings -> webhooks -> add webhook


#NOTE: watch the source
flux get sources git 2-demo-source-git-bx-game-app --watch



```

### alert and provider

https://fluxcd.io/flux/components/notification/

send alert to slack channel using notification controller

https://fluxcd.io/flux/components/notification/providers/
```shell
#setup the slack workspace and channel
#create slack app and add the slack app to workspace
#token from slack app api

#create secret for slack token
kubectl -n flux-system create secret generic slack-bot-token \
--from-literal=token=

#create slack provider
flux create alert-provider notification-provider-slack \
  --type=slack \
  --channel=flux-alerts-channel \
  --username=flux-bot \
  --secret-ref=slack-bot-token \
  --address=https://slack.com/api/chat.postMessage \
  --export > notification-provider-slack.yaml
  
  
 #filter the event using alert
flux create alert notification-alert-slack \
    --event-severity info \
    --provider-ref notification-provider-slack \
    --event-source "Kustomization/*" \
    --event-source "HelmRelease/*" \
    --event-source "GitRepository/*" \
    --event-source "ImageRepository/*" \
    --event-source "Bucket/*" \
    --event-source "HelmRepository/*" \
    --event-source "ImagePolicy/*" \
    --event-source "ImageUpdateAutomation/*" \
    --event-source "OCIRepository/*" \
    --event-source "HelmChart/*" \
    --export > notification-alert-slack.yaml
    
#reconcile the source
flux reconcile source git flux-system
flux reconcile kustomization 2-demo-kustomization-bx-game-app
flux get alert
flux get alert-providers
```

## flux monitoring and user interface

### install prometheus and grafana

```shell
#create source git
flux create source git monitoring-source-prometheus-stack \
  --interval=30m \
  --url=https://github.com/fluxcd/flux2 \
  --branch=main \
  --export > monitoring-source-prometheus-stack.yaml

#create kustomization
flux create kustomization monitoring-kustomization-prometheus-stack \
  --interval=1h \
  --source=monitoring-source-prometheus-stack \
  --path="./manifests/monitoring/kube-prometheus-stack" \
  --export > monitoring-kustomization-prometheus-stack.yaml
  
#update the svc type to NodePort
k -n monitoring edit svc kube-prometheus-stack-prometheus

#update the svc type to NodePort
k -n monitoring edit svc kube-prometheus-stack-grafana

#access the prometheus and grafana in user interface
k -n monitoring get svc
https://localhost:31549/
https://localhost:31793/

#extract the grafana password
kubectl get secret --namespace monitoring kube-prometheus-stack-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo

```

### configure prometheus to scrape the metrics from flux and visualize in grafana

```shell

#create kustomization 

flux create kustomization monitoring-ocnfig \
  --depends-on=monitoring-kustomization-prometheus-stack \
  --interval=1h \
  --prune=true \
  --source=monitoring-source-prometheus-stack \
  --path="./manifests/monitoring/monitoring-config" \
  --export > monitoring-ocnfig.yaml

#reconcile the source
flux reconcile source git flux-system 

#verify the podmonitors
k -n monitoring get podmonitors

#install weave gitops

# ref https://docs.gitops.weave.works/docs/open-source/getting-started/install-OSS/

brew tap weaveworks/tap
brew install weaveworks/tap/gitops

gitops --version

gitops create dashboard ww-gitops \
  --password=pass
  
#access the gitops dashboard
gitops dashboard
k get svc -n flux-system
#expose service as nodeport
k -n flux-system edit svc ww-gitops-weave-gitops

#access the gitops dashboard
https://localhost:31281/
```

## flux uninstall

```text
flux uninstall --namespace=flux-system

Are you sure you want to delete Flux and its custom resource definitions: y
► deleting components in flux-system namespace
✔ Deployment/flux-system/helm-controller deleted 
✔ Deployment/flux-system/image-automation-controller deleted 
✔ Deployment/flux-system/image-reflector-controller deleted 
✔ Deployment/flux-system/kustomize-controller deleted 
✔ Deployment/flux-system/notification-controller deleted 
✔ Deployment/flux-system/source-controller deleted 
✔ Service/flux-system/notification-controller deleted 
✔ Service/flux-system/receiver deleted 
✔ Service/flux-system/source-controller deleted 
✔ Service/flux-system/webhook-receiver deleted 
✔ NetworkPolicy/flux-system/allow-egress deleted 
✔ NetworkPolicy/flux-system/allow-scraping deleted 
✔ NetworkPolicy/flux-system/allow-webhooks deleted 
✔ ServiceAccount/flux-system/helm-controller deleted 
✔ ServiceAccount/flux-system/image-automation-controller deleted 
✔ ServiceAccount/flux-system/image-reflector-controller deleted 
✔ ServiceAccount/flux-system/kustomize-controller deleted 
✔ ServiceAccount/flux-system/notification-controller deleted 
✔ ServiceAccount/flux-system/source-controller deleted 
✔ ClusterRole/crd-controller-flux-system deleted 
✔ ClusterRole/flux-edit-flux-system deleted 
✔ ClusterRole/flux-view-flux-system deleted 
✔ ClusterRoleBinding/cluster-reconciler-flux-system deleted 
✔ ClusterRoleBinding/crd-controller-flux-system deleted 
► deleting toolkit.fluxcd.io finalizers in all namespaces
✔ GitRepository/flux-system/2-demo-source-git-bx-game-app finalizers deleted 
✔ GitRepository/flux-system/3-demo-source-git-bx-game-app finalizers deleted 
✔ GitRepository/flux-system/5-demo-source-helm-git-bx-game-app finalizers deleted 
✔ GitRepository/flux-system/8-demo-source-git-bx-game-app finalizers deleted 
✔ GitRepository/flux-system/flux-system finalizers deleted 
✔ GitRepository/flux-system/infra-source-git finalizers deleted 
✔ GitRepository/flux-system/monitoring-source-prometheus-stack finalizers deleted 
✔ OCIRepository/flux-system/10-demo-source-oci-dx-game-app finalizers deleted 
✔ OCIRepository/flux-system/7-demo-infra-source-oci-dx-game-app-770 finalizers deleted 
✔ HelmRepository/flux-system/6-demo-source-helm-repo-bx-game-app finalizers deleted 
✔ HelmRepository/flux-system/sealed-secrets finalizers deleted 
✔ HelmRepository/flux-system/ww-gitops finalizers deleted 
✔ HelmRepository/monitoring/prometheus-community finalizers deleted 
✔ HelmChart/flux-system/flux-system-5-demo-helmrelease-bx-game-app finalizers deleted 
✔ HelmChart/flux-system/flux-system-6-demo-helmrelease-bx-game-app finalizers deleted 
✔ HelmChart/flux-system/flux-system-sealed-secrets finalizers deleted 
✔ HelmChart/flux-system/flux-system-ww-gitops finalizers deleted 
✔ HelmChart/monitoring/monitoring-kube-prometheus-stack finalizers deleted 
✔ Bucket/flux-system/4-demo-source-minio-s3-bucket-bx-game-app finalizers deleted 
✔ Kustomization/flux-system/10-demo-kustomization-oci-dx-game-app finalizers deleted 
✔ Kustomization/flux-system/2-demo-kustomization-bx-game-app finalizers deleted 
✔ Kustomization/flux-system/3-demo-kustomization-bx-game-app finalizers deleted 
✔ Kustomization/flux-system/4-demo-kustomization-minio-s3-bucket-bx-game-app finalizers deleted 
✔ Kustomization/flux-system/7-demo-infra-kustomization-oci-dx-game-app-770 finalizers deleted 
✔ Kustomization/flux-system/8-demo-kustomization-git-bx-game-app finalizers deleted 
✔ Kustomization/flux-system/flux-system finalizers deleted 
✔ Kustomization/flux-system/infra-security-kustomize-git-sealed-secrets finalizers deleted 
✔ Kustomization/flux-system/monitoring-kustomization-prometheus-stack finalizers deleted 
✔ Kustomization/flux-system/monitoring-ocnfig finalizers deleted 
✔ HelmRelease/flux-system/5-demo-helmrelease-bx-game-app finalizers deleted 
✔ HelmRelease/flux-system/6-demo-helmrelease-bx-game-app finalizers deleted 
✔ HelmRelease/flux-system/sealed-secrets finalizers deleted 
✔ HelmRelease/flux-system/ww-gitops finalizers deleted 
✔ HelmRelease/monitoring/kube-prometheus-stack finalizers deleted 
✔ Alert/flux-system/notification-alert-slack finalizers deleted 
✔ Provider/flux-system/notification-provider-slack finalizers deleted 
✔ Receiver/flux-system/github-webhook-receiver finalizers deleted 
✔ ImagePolicy/flux-system/8-demo-image-policy-bx-game-app finalizers deleted 
✔ ImageRepository/flux-system/8-demo-image-repository-bx-game-app finalizers deleted 
✔ ImageUpdateAutomation/flux-system/8-demo-image-update-bx-game-app finalizers deleted 
► deleting toolkit.fluxcd.io custom resource definitions
✔ CustomResourceDefinition/alerts.notification.toolkit.fluxcd.io deleted 
✔ CustomResourceDefinition/buckets.source.toolkit.fluxcd.io deleted 
✔ CustomResourceDefinition/gitrepositories.source.toolkit.fluxcd.io deleted 
✔ CustomResourceDefinition/helmcharts.source.toolkit.fluxcd.io deleted 
✔ CustomResourceDefinition/helmreleases.helm.toolkit.fluxcd.io deleted 
✔ CustomResourceDefinition/helmrepositories.source.toolkit.fluxcd.io deleted 
✔ CustomResourceDefinition/imagepolicies.image.toolkit.fluxcd.io deleted 
✔ CustomResourceDefinition/imagerepositories.image.toolkit.fluxcd.io deleted 
✔ CustomResourceDefinition/imageupdateautomations.image.toolkit.fluxcd.io deleted 
✔ CustomResourceDefinition/kustomizations.kustomize.toolkit.fluxcd.io deleted 
✔ CustomResourceDefinition/ocirepositories.source.toolkit.fluxcd.io deleted 
✔ CustomResourceDefinition/providers.notification.toolkit.fluxcd.io deleted 
✔ CustomResourceDefinition/receivers.notification.toolkit.fluxcd.io deleted 
✔ Namespace/flux-system deleted 
✔ uninstall finished
```
