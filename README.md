# kube playground

This is an experiment with k3s and metallb that runs in Vagrant.

- [kube playground](#kube-playground)
  - [Usage](#Usage)
  - [Doc Links](#Doc-Links)
  - [Sample Apps](#Sample-Apps)
    - [Argo CD](#Argo-CD)
    - [OpenFaaS](#OpenFaaS)

## Usage

Helm is preinstalled but needs to be setup:

```bash
kubectl -n kube-system create sa tiller \
  && kubectl create clusterrolebinding tiller \
  --clusterrole cluster-admin \
  --serviceaccount=kube-system:tiller

helm init --skip-refresh --upgrade --service-account tiller
```

## Doc Links

- https://argoproj.github.io/argo-cd/getting_started/
- https://github.com/openfaas/workshop/blob/master/lab1b.md

## Sample Apps

### Argo CD

These are the steps used to install Argo CD as of 12 July 2019:

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

These are the steps used to install OpenFaaS as of 12 July 2019 (be sure you have run the Helm install above):

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

Run the following from your local machine

```bash
export OPENFAAS_PW=<the password displayed above>
export OPENFAAS_URL=http://<OPENFAAS_GATEWAY_EXTERNAL>:8080 # the get svc command above will show you the address
echo -n $OPENFAAS_PW | faas-cli login -g $OPENFAAS_URL -u admin --password-stdin
```

You can also log into the web interface via http://<OPENFAAS_GATEWAY_EXTERNAL>:8080

OpenFaaS can be removed via these commands if you no longer wish to run it.

```bash
helm delete --purge openfaas
kubectl delete namespace openfaas openfaas-fn
```
