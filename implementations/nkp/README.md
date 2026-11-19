# NKP Specifics:

# Changes
- 12-Aug-2026:  -Initial Commit


## Flagger Install Steps:

```
helm upgrade -i flagger flagger/flagger \
--namespace kommander \
--set metricsServer=http://kube-prometheus-stack-prometheus:9090
```

```
kubectl -n kommander logs deploy/flagger -f | jq .msg
```

## Basic Test

### Create podinfo
```
kubectl apply -k https://github.com/fluxcd/flagger//kustomize/podinfo?ref=main
```

### LoadTesting Service
```
kubectl apply -k https://github.com/fluxcd/flagger//kustomize/tester?ref=main
```

### Add canary even tho b/g?
```
kubectl apply -f ./podinfo-canary.yaml
```

### Trigger a deployment.  New?
```
kubectl -n test set image deployment/podinfo \
podinfod=ghcr.io/stefanprodan/podinfo:6.0.1
```

```
kubectl -n test describe canary/podinfo
```

### Clean-Up

```
kubectl delete -k https://github.com/fluxcd/flagger//kustomize/podinfo?ref=main
```

```
kubectl delete -k https://github.com/fluxcd/flagger//kustomize/tester?ref=main
```
