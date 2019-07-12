# kube playground

This is an experiment with k3s and metallb that runs in Vagrant.

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
