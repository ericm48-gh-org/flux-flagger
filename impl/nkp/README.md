# NKP Specifics:

# Changes
- 19-Nov-2026:  -Updated.
- 12-Aug-2026:  -Initial Commit

## Assumptions/Pre-Requesites:

1) Current NKP Cluster (> v2.17.0) setup.  
2) FULL Access to NKP Mgmt-Cluster via KubeConfig file.
3) Flagger & Target App [ petclinic ] will be installed on mgmt-cluster for illustrative purposes only.  This is not a production configuration.   This is to allow ease of access to Prometheus Metrics, etc.
4) K8s-CLI [ kubectl ], version >1.35 installed operational.
5) kubectl commands are often aliased by `k`.
5) K9s version > 1.50.x installed.
6) Some commands & paths below will vary based off of your implementation.  Below are mine based off of an NKP-v2.18.1 implementation.



## Steps:

1) Setup kubectl access to mgmt-cluster (example):

```
export KUBECONFIG='/home/ubuntu/nkp.conf'
```

2) Add flagger Helm repo:
```
helm repo add flagger https://flagger.app
```

3) Update helm repo:
```
helm repo update
```

4) Helm install Flagger to `kommander` namespace. [ Yes, it has to be kommander NS to have Prometheus reachable ]:
```
helm upgrade -i flagger flagger/flagger \
--namespace kommander \
--set cmd.timeout=30m \
--set metricsServer=http://kube-prometheus-stack-prometheus:9090  
```
If you need to remove Flagger, use:
```
helm -n kommander  uninstall flagger
```

5) Verify Flagger started, check logs:
```
k -n kommander logs deploy/flagger -f | jq .msg
```
You should see activity.


6) Create the `test` namespace:
```
k create ns test
```

7) Helm Install the Flagger-LoadTester [ your timeout may vary ]:
```
helm upgrade -i flagger-loadtester flagger/loadtester \
--namespace=test \
--set cmd.timeout=1h
```

If you need to uninstall Flagger-LoadTester, use:
```
helm -n test uninstall flagger-loadtester 
```

8) Apply the CRD:
```
cd ./flux-flagger/crd
k apply -f ./crd-traefik-ingressroute.yaml
```

9)
```

```


















### X) Flagger Install Steps:

```
helm upgrade -i flagger flagger/flagger \
--namespace kommander \
--set metricsServer=http://kube-prometheus-stack-prometheus:9090
```

```
k -n kommander logs deploy/flagger -f | jq .msg
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
