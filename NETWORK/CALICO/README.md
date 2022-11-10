# install calico using helm

```
helm repo add projectcalico https://docs.projectcalico.org/charts
```
# install tigera

```
helm install calico projectcalico/tigera-operator --version v3.21.4
```