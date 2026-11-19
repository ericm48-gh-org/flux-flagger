# Minikube Specifics:

# Changes
- 12-Aug-2026:  -Initial Commit


# Minikube Cluster7

## Handy Alias:
```
alias hru='helm repo update'
```


## Flux Install:
```
helm repo add fluxcd-community https://fluxcd-community.github.io/helm-charts
```

```
helm repo update
```

```
helm install flux2 fluxcd-community/flux2 \
  --namespace flux-system \
  --create-namespace
```

## Flagger Install: 

```
helm upgrade -i flagger flagger/flagger \
--namespace flagger-system \
--set prometheus.install=true \
--set meshProvider=kubernetes \
--create-namespace
```

## Basic Test:

```
kubectl create ns test
```

```
kubectl apply -k https://github.com/fluxcd/flagger//kustomize/podinfo?ref=main
```

```
kubectl apply -k https://github.com/fluxcd/flagger//kustomize/tester?ref=main
```

```
k apply -f podinfo-canary-bg.yaml 
```

## Fails b/c of lack of Prom...OK...

## 1. Add the official Prometheus Community Helm repository
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```
## 2. Update your local Helm chart registry
```
helm repo update
```

## 3. Create a namespace and install the unified Prometheus and Grafana stack
```
helm install prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace
```

## kube-prometheus-stack has been installed. Check its status by running:
```
kubectl --namespace monitoring get pods -l "release=prometheus-stack"
```

## Get Grafana 'admin' user password by running:

```
kubectl --namespace monitoring get secrets prometheus-stack-grafana -o jsonpath="{.data.admin-password}" | base64 -d ; echo
```

## Access Grafana local instance:
```
export POD_NAME=$(kubectl --namespace monitoring get pod -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=prometheus-stack" -oname)
```

```
kubectl --namespace monitoring port-forward $POD_NAME 3000 &
```

## Get your grafana admin user password by running:

```
kubectl get secret --namespace monitoring -l app.kubernetes.io/component=admin-secret -o jsonpath="{.items[0].data.admin-password}" | base64 --decode ; echo
```


Visit https://github.com/prometheus-operator/kube-prometheus for instructions on how to create & configure Alertmanager and Prometheus instances using the Operator.

## Add loki

```
helm repo add grafana https://grafana.github.io/helm-charts
```

```
helm repo update
```

```
helm upgrade --install loki-stack grafana/loki-stack \
  --create-namespace \
  --namespace loki-system \
  --set grafana.enabled=true
```

```
helm -n loki-system uninstall loki-stack
```

## Next trying adding promtail...
```
helm upgrade --install loki-stack grafana/loki-stack \
  --create-namespace \
  --namespace loki-system \
  --set grafana.enabled=true \
  --set promtail.enabled=true
```  

## Loki is crashing b/c of too-many open files... GRRRR!!!

## This FIXES it!
```
echo "fs.inotify.max_user_instances=512" | sudo tee -a /etc/sysctl.conf
```

```
echo "fs.inotify.max_user_watches=1048576" | sudo tee -a /etc/sysctl.conf
```

```
sudo sysctl -p
```

## Now..it works... B/G Progressive works on minikube.. 

