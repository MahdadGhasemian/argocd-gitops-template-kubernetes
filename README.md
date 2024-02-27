![argocd-gitops](./.images/argocd-gitops.png)

# Gitlab Repository

Because of using the gitlab container registry and also gitlab group feature, I've created all repositories inside the Gitlab, you can find all resources inside the gitlab repository here:
[https://gitlab.com/argocd-gitops-template](https://gitlab.com/argocd-gitops-template)

![gitlab-tree](./.images/gitlab-tree.png)

# Requirements

- [Kubernetes](https://www.redhat.com/en/topics/containers/what-is-kubernetes) cluster :
  In order to have a kubernetes cluster we have several options:
  1- Create a kubernetes service on a cloud provider like [AWS](https://aws.amazon.com/)
  2- Create a kubernetes service on the [Hetzner Cloud](https://www.hetzner.com/cloud/)
  I've created a template to setup a highly avalable kubernetes on the Hetzner Cloud here [kubernetes-hetzner-lac-template](https://github.com/MahdadGhasemian/kubernetes-hetzner-lac-templatev) and described it here [Kubernetus Cluster](https://medium.com/@mahdad.ghasemian/setting-up-a-highly-available-kubernetus-cluster-k3s-on-hetzner-cloud-with-terraform-7a409a7a8528)
  3- [Minikube](https://minikube.sigs.k8s.io/docs/) as a local k8s development
  4- [K3S](https://k3s.io/) (local and production)
  5- [K3D](https://k3d.io/v5.6.0/) as a local k3s development
- [Kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl) to interact with the Kubernetes cluster
- [Helm](https://helm.sh/docs/intro/install/) to install packegs on our kubernetes
- [Docker](https://www.docker.com/) to install k3d

## What is Minikube ?

Minikube is local Kubernetes, focusing on making it easy to learn and develop for Kubernetes.

## What is K3S ?

K3s is a highly available, certified Kubernetes distribution designed for production workloads in unattended, resource-constrained, remote locations or inside IoT appliances.

![K3s](./.images/how-it-works-k3s-revised.svg)

## What is K3D ?

k3d is a lightweight wrapper to run k3s (Rancher Lab’s minimal Kubernetes distribution) in docker.
k3d makes it very easy to create single- and multi-node k3s clusters in docker, e.g. for local development on Kubernetes.

![k3d](./.images/k3d_logo_black_blue.svg)

## Conclusion

I'm going to install and use k3d for this lecture because of:

- It's free
- It's easy to use
- It could be installed on docker and I assume all users have installed the docker on thier systems (linux, windows or mac)

# Install K3d on docker and config a k3s cluster

To install k3d according its [release page](https://k3d.io/v5.6.0/#releases) there are several methods to install, so I choose install current latest version with its script:

```bash
wget -q -O - https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
k3d version
# k3d version v5.6.0
# k3s version v1.27.4-k3s1 (default)
```

after installing k3d, now we can run a k3d cluster with 1 server as contoller and 2 server as agent

```bash
k3d cluster create --servers 1 --agents 2
# INFO[0000] portmapping '443:443' targets the loadbalancer: defaulting to [servers:*:proxy agents:*:proxy]
# INFO[0000] portmapping '80:80' targets the loadbalancer: defaulting to [servers:*:proxy agents:*:proxy]
# INFO[0000] Prep: Network
# INFO[0000] Created network 'k3d-k3s-default'
# INFO[0000] Created image volume k3d-k3s-default-images
# INFO[0000] Starting new tools node...
# INFO[0000] Starting Node 'k3d-k3s-default-tools'
# INFO[0001] Creating node 'k3d-k3s-default-server-0'
# INFO[0001] Creating node 'k3d-k3s-default-agent-0'
# INFO[0001] Creating node 'k3d-k3s-default-agent-1'
# INFO[0001] Creating LoadBalancer 'k3d-k3s-default-serverlb'
# INFO[0002] Using the k3d-tools node to gather environment information
# INFO[0002] Starting new tools node...
# INFO[0002] Starting Node 'k3d-k3s-default-tools'
# INFO[0003] Starting cluster 'k3s-default'
# INFO[0003] Starting servers...
# INFO[0003] Starting Node 'k3d-k3s-default-server-0'
# INFO[0007] Starting agents...
# INFO[0007] Starting Node 'k3d-k3s-default-agent-1'
# INFO[0007] Starting Node 'k3d-k3s-default-agent-0'
# INFO[0010] Starting helpers...
# INFO[0011] Starting Node 'k3d-k3s-default-serverlb'
# INFO[0017] Injecting records for hostAliases (incl. host.k3d.internal) and for 5 network members into CoreDNS configmap...
# INFO[0019] Cluster 'k3s-default' created successfully!
# INFO[0019] You can now use it like this:
# kubectl cluster-info
```

now we can read the cluster info

```bash
kubectl cluster-info
# Kubernetes control plane is running at https://0.0.0.0:35239
# CoreDNS is running at https://0.0.0.0:35239/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
# Metrics-server is running at https://0.0.0.0:35239/api/v1/namespaces/kube-system/services/https:metrics-server:https/proxy
```

and if you read your kubectl's contexts details, it should be listed as default (k3d-k3s-default)

```bash
kubectl config get-contexts
# CURRENT   NAME              CLUSTER           AUTHINFO                NAMESPACE
# *         k3d-k3s-default   k3d-k3s-default   admin@k3d-k3s-default
#           k3s               k3s               k3s
#           minikube          minikube          minikube                default
```

```bash
kubectl get all
# NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
# service/kubernetes   ClusterIP   10.43.0.1    <none>        443/TCP   4m56s
```

# Configure

## ArgoCD

### Install argocd

```bash
kubectl create namespace argocd
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm install argo-cd argo/argo-cd --namespace argocd --version 6.3.1 \
--set configs.params.server\.insecure=true \
--set dex.enabled=true
# NAME: argo-cd
# LAST DEPLOYED: Mon Feb 26 16:19:07 2024
# NAMESPACE: argocd
# STATUS: deployed
# REVISION: 1
# TEST SUITE: None
# NOTES:
# In order to access the server UI you have the following options:

# 1. kubectl port-forward service/argo-cd-argocd-server -n argocd 8080:443

#     and then open the browser on http://localhost:8080 and accept the certificate

# 2. enable ingress in the values file `server.ingress.enabled` and either
#       - Add the annotation for ssl passthrough: https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/#option-1-ssl-passthrough
#       - Set the `configs.params."server.insecure"` in the values file and terminate SSL at your ingress: https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/#option-2-multiple-ingress-objects-and-hosts


# After reaching the UI the first time you can login with username: admin and the random password generated during the installation. You can find the password by running:

# kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# (You should delete the initial secret afterwards as suggested by the Getting Started Guide: https://argo-cd.readthedocs.io/en/stable/getting_started/#4-login-using-the-cli)

```

### Port forward argocd to access it and read its admin password to login

```bash
kubectl port-forward service/argo-cd-argocd-server -n argocd 8080:443
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

## ArgoCD Image Updater

### Install argocd image updater

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm install argocd-image-updater argo/argocd-image-updater --namespace argocd --version 0.9.1 \
--set-json='config.registries=[{"name":"Gilab Registry", "api_url":"https://registry.gitlab.com", "prefix":"registry.gitlab.com", "ping":true, "credentials":"secret:argocd/gitlabregistry#token", "limit":20, "default":true, "defaultns":"library"}]'
```

## Gitlab access

### Access to the deploy project to write latest changes

- Create an ed25519 key on your local pc

```bash
ssh-keygen -t ed25519 -f ~/.ssh/argocd-gitops-template-ed25519
```

- Go to the deploy repository (for our group: https://gitlab.com/argocd-gitops-template/infrastructure/deploy)
- Settings -> Repository -> Deploy keys -> Add new key
  ![gitlab-setting-repository-deploy-keys](./.images/gitlab-setting-repository-deploy-keys.png)
- Copy the public key:
  - Write a title
  - Copy your public key on the key part
  ```bash
    cat ~/.ssh/argocd-gitops-template-ed25519.pub
  ```
  - Check the `Grant write permissions to this key`
    ![gitlab-setting-repository-deploy-keys-fill](./.images/gitlab-setting-repository-deploy-keys-fill.png)
- Edit the git-repo-secret.yaml:

  - Change the `stringData.url` (alter to your deploy repository address)
  - Add the private key into the `stringData.sshPrivateKey`:

    ```bash
      cat ~/.ssh/argocd-gitops-template-ed25519
    ```

    ```yaml
    # git-repo-secret.yaml
    ---
    apiVersion: v1
    kind: Secret
    metadata:
      name: argocd-gitops-template
      namespace: argocd
      labels:
        argocd.argoproj.io/secret-type: repository
    stringData:
      url: git@gitlab.com:argocd-gitops-template/infrastructure/deploy.git
      sshPrivateKey: |
        -----BEGIN OPENSSH PRIVATE KEY-----
        PASTE_YOUR_PRIVATE_KEY_HERE
        -----END OPENSSH PRIVATE KEY-----
      insecure: "false"
      enableLfs: "true"
    ```

- Add key to the kubernetes secrets (argocd)

```bash
  kubectl apply -f git-repo-secret.yaml
```

### Access to the group registery read

- Go to the Group (ArgoCD GitOps Template)
- Settings -> Repository -> Deploy tokens -> Add token
- Create a new deploy token:
  - Name: read_registery_and_package
  - Scopes: read_registry and read_package_registry
  - username: k8s
    ![gitlab-registery-read-access](./.images/gitlab-registery-read-access.png)
- Create secret.yaml file and replace the token:

  - Change the `stringData.token`

    ```yaml
    # secret.yaml
    ---
    apiVersion: v1
    kind: Secret
    metadata:
      name: gitlabregistry
      namespace: argocd
    type: Opaque
    stringData:
      token: k8s:REPLACE_WITH_YOUR_TOKEN
    ```

- Add key to the kubernetes secrets (argocd)

```bash
kubectl apply -f secret.yaml
```

# Project structure

I've created a group for having all side of our project in it.

```
Argocd Gitops Template
│
└───Backend
│   │   Nodejs Simple
│
└───Frontend
│   │   Reactjs Simple
│
└───Infrastructure
    │   Deploy
    │   Kubernetes
```

| Project        | Description                                                                                                                 |
| -------------- | --------------------------------------------------------------------------------------------------------------------------- |
| Nodejs Simple  | A simple nodejs todo app as backend                                                                                         |
| Reactjs Simple | A simple reactjs todo app as frontend                                                                                       |
| Deploy         | We keep all the devops role about the our backend and frontend services and the argocd synks itself with this repo all time |
| Kubernetes     | We only have keep our initial commands and also this README file                                                            |

## Backend and Frontend ci files

There are 3 branches for backend and frontend projects:

- developing
- stage
- main

We're going to deploy stage branch as staging and main branch as a production service.

In the `gitlab.ci` file we have to separate pipeline to make images for staging and production.
It will automatically generate an image from out backend or frontend projects and put them inside the gitlab container registery

![gitlab-container-registery](./.images/gitlab-container-registery.png)
![gitlab-container-registery-detail](./.images/gitlab-container-registery-detail.png)

```yaml
default:
  image: docker:20.10.16
  services:
    - docker:20.10.16-dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY

stages:
  - build_and_release

variables:
  CONTAINER_BUILD_IMAGE: $CI_REGISTRY_IMAGE/$CI_COMMIT_REF_NAME:v0.1.$CI_PIPELINE_IID
  CONTAINER_RELEASE_IMAGE: $CI_REGISTRY_IMAGE/$CI_COMMIT_REF_NAME:latest

stage-release:
  stage: build_and_release
  script:
    - >
      docker build
      --pull
      -t $CONTAINER_BUILD_IMAGE .
    - docker push $CONTAINER_BUILD_IMAGE
    - docker tag $CONTAINER_BUILD_IMAGE $CONTAINER_RELEASE_IMAGE
    - docker push $CONTAINER_RELEASE_IMAGE
  only:
    - stage

main-release:
  stage: build_and_release
  script:
    - >
      docker build
      --pull
      -t $CONTAINER_BUILD_IMAGE .
    - docker push $CONTAINER_BUILD_IMAGE
    - docker tag $CONTAINER_BUILD_IMAGE $CONTAINER_RELEASE_IMAGE
    - docker push $CONTAINER_RELEASE_IMAGE
  only:
    - main
```

When a new version is generated, the argocd image updater will be informed and then start to create a new file inside the `deploy` repository to tell the argocd there is new update and then the argocd will refresh the app and is synked with new update.

An example of file version

![argocd-image-updater-git-method](./.images/argocd-image-updater-git-method.png)

# Run Applications

## Create Namespaces

```bash
kubectl create namespace staging-nodejs-simple
kubectl create namespace staging-reactjs-simple
```

```bash
kubectl create namespace production-nodejs-simple
kubectl create namespace production-reactjs-simple
```

## Create registery credential

```bash
kubectl create secret docker-registry registrycredential -n argocd \
    --docker-server=registry.gitlab.com \
    --docker-username=k8s \
    --docker-password=REPLACE_WITH_YOUR_TOKEN
```

```bash
kubectl create secret docker-registry registrycredential -n staging-nodejs-simple \
    --docker-server=registry.gitlab.com \
    --docker-username=k8s \
    --docker-password=REPLACE_WITH_YOUR_TOKEN

kubectl create secret docker-registry registrycredential -n staging-reactjs-simple \
    --docker-server=registry.gitlab.com \
    --docker-username=k8s \
    --docker-password=REPLACE_WITH_YOUR_TOKEN
```

```bash
kubectl create secret docker-registry registrycredential -n production-nodejs-simple \
    --docker-server=registry.gitlab.com \
    --docker-username=k8s \
    --docker-password=REPLACE_WITH_YOUR_TOKEN

kubectl create secret docker-registry registrycredential -n production-reactjs-simple \
    --docker-server=registry.gitlab.com \
    --docker-username=k8s \
    --docker-password=REPLACE_WITH_YOUR_TOKEN
```

## Create Applications

```bash
kubectl apply -f application-staging.yaml
kubectl apply -f application-production.yaml
```

![argocd-applications](./.images/argocd-applications.png)
![argocd-detail](./.images/argocd-detail.png)

## Note

The ui projects need to know what is the api base url
so you should add the `APP_BASE_URL_STAGING` and `APP_BASE_URL_PRODUCTION` environment inside the `Frontend` group and then merge the project from developing into stage and from stage into main branch.

![gitlab-environment-1](./.images/gitlab-environment-1.png) ![gitlab-environment-2](./.images/gitlab-environment-2.png)
