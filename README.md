# kube playground

This is my place to keep things I play with in [k3s](https://k3s.io/) via [MetalLB](https://metallb.universe.tf/). This setup is all designed to be run in Vagrant on a laptop.

- [kube playground](#kube-playground)
  - [Usage](#usage)
    - [Helm setup](#helm-setup)
  - [Build notes](#build-notes)
    - [K3s in Vagrant](#k3s-in-vagrant)
    - [K3s and MetalLB](#k3s-and-metallb)
    - [Additional K3s master flags](#additional-k3s-master-flags)
    - [Helm & k3s's kube config](#helm--k3ss-kube-config)
  - [Sample Apps](#sample-apps)
    - [Argo CD](#argo-cd)
    - [OpenFaaS](#openfaas)
    - [Polaris](#polaris)

## Usage

To get started you just need [Vagrant](https://www.vagrantup.com/) installed. Once you have that run the following:

```bash
vagrant up
```

That will spin up a master node and two worker nodes by default. Once they are up you can log into the master like so:

```bash
vagrant ssh master
```

### Helm setup

Helm is preinstalled but needs to be setup:

```bash
kubectl -n kube-system create sa tiller \
  && kubectl create clusterrolebinding tiller \
  --clusterrole cluster-admin \
  --serviceaccount=kube-system:tiller

helm init --skip-refresh --upgrade --service-account tiller
```

## Build notes

### K3s in Vagrant

K3s needs some extra configuration to work correctly inside of Vagrant. In particular, these settings were required:

- `--node-ip=` needs to be set to the IP of eth1
- `--flannel-iface=eth1` has to be set so that Flannel will talk over the interface that works without voodoo (Vagrant does special things with the first interface)

### K3s and MetalLB

Since we are using MetalLB as our load balancer we need to pass the master the install flags `--no-deploy=servicelb`

### Additional K3s master flags

And, just because of the way this setup uses k3s, we need a couple of more flags:

- `--no-deploy=traefik` is so that we can choose our own ingress / proxy later
- `--write-kubeconfig-mode 644` is so that the user `vagrant` can utilize the config laid down by the installer

### Helm & k3s's kube config

K3s ships with a special version of `kubectl` that knows where to find the config laid down by the installer but helm know nothing of this. To make life simpler we make everyone aware of it by default by creating `/etc/profile.d/k3s-kubeconfig.sh` like so:

```bash
echo "export KUBECONFIG='/etc/rancher/k3s/k3s.yaml'" > /etc/profile.d/k3s-kubeconfig.sh
```

## Sample Apps

### Argo CD

Docs used: https://argoproj.github.io/argo-cd/getting_started/

Installation process as of 12 July 2019:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
kubectl get svc -o wide argocd-server -n argocd
kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o name | cut -d'/' -f 2
```

Once that is finished, run these commands on your local machine:

```bash
argocd login <ARGOCD_SERVER> # this is shown via the get svc line above
argocd account update-password # set to something easy to remember

argocd app create guestbook \
  --repo https://github.com/argoproj/argocd-example-apps.git \
  --path guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default

argocd app sync guestbook
```

You can also visit the web interface via http://<ARGOCD_SERVER>

### OpenFaaS

Docs used: https://github.com/openfaas/workshop/blob/master/lab1b.md

Installation process as of 12 July 2019:

First, be sure you have run the Helm install above.

```bash
kubectl apply -f https://raw.githubusercontent.com/openfaas/faas-netes/master/namespaces.yml
helm repo add openfaas https://openfaas.github.io/faas-netes/

# generate a random password
PASSWORD=$(head -c 12 /dev/urandom | shasum| cut -d' ' -f1)
echo $PASSWORD > gateway-password.txt

kubectl -n openfaas create secret generic basic-auth \
--from-literal=basic-auth-user=admin \
--from-literal=basic-auth-password="$PASSWORD"

helm repo update \
 && helm upgrade openfaas --install openfaas/openfaas \
    --namespace openfaas  \
    --set basic_auth=true \
    --set serviceType=LoadBalancer \
    --set functionNamespace=openfaas-fn
```

At this point, OpenFaaS should be coming up. You can check its status via these commands:

```bash
kubectl --namespace=openfaas get deployments -l "release=openfaas, app=openfaas"
kubectl get pods -n openfaas
```

Once all the components show to be up you can continue. First, you need the password set earlier:

```bash
kubectl get svc -o wide gateway-external -n openfaas
cat gateway-password.txt
```

You can now log into the web interface via http://<OPENFAAS_GATEWAY_EXTERNAL>:8080

You can also run the following from your local machine to use the faas-cli:

```bash
export OPENFAAS_PW=<the password displayed above>
export OPENFAAS_URL=http://<OPENFAAS_GATEWAY_EXTERNAL>:8080 # the get svc command above will show you the address
echo -n $OPENFAAS_PW | faas-cli login -g $OPENFAAS_URL -u admin --password-stdin
```

OpenFaaS can be removed via these commands if you no longer wish to run it.

```bash
helm delete --purge openfaas
kubectl delete namespace openfaas openfaas-fn
```

### Polaris

Docs used: https://github.com/FairwindsOps/polaris

Installation process as of 25 July 2019:

```bash
# Install Polaris
kubectl apply -f https://github.com/reactiveops/polaris/releases/latest/download/dashboard.yaml

# Switch from ClusterIP to LoadBalancer
kubectl patch svc polaris-dashboard -n polaris -p '{"spec": {"type": "LoadBalancer"}}'

# Get the address needed to accss Polaris
kubectl get svc -o wide polaris-dashboard -n polaris
```
