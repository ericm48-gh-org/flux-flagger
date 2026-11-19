# NKP Specifics:

# Changes
- 19-Nov-2026:  -Updated.
- 12-Aug-2026:  -Initial Commit

## Assumptions/Pre-Requesites:

1) Current NKP Cluster (> v2.17.0) setup.  
2) FULL Access to NKP Mgmt-Cluster via KubeConfig file.
3) Flagger & Target App [ petclinic ] will be installed on mgmt-cluster for illustrative purposes only.  This is not a production configuration.   This is to allow ease of access to Prometheus Metrics, etc.
4) K8s-CLI [ kubectl ], version >1.35 installed, operational.
5) KNS-CLI [ Kubernetes Namespace Utility ] installed, operational.
6) kubectl commands are often aliased by `k`.
7) K9s version > 1.50.x installed, operational.
8) Some commands & paths below will vary based off of your implementation.  Below are mine based off of an NKP-v2.18.1 implementation.
9) When petclinic is deployed it takes approximately 60-65 seconds for the application to start.  The default deployment consists of 2 replicas.


## Steps:

1) Setup kubectl access to mgmt-cluster (example):

```
export KUBECONFIG='/home/ubuntu/nkp.conf'
```

2) Add Flagger Helm repo:
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

7) Set the default kubernetes namespace to `test`:
```
kns test
```

8) Helm Install the Flagger-LoadTester [ your timeout may vary ]:
```
helm upgrade -i flagger-loadtester flagger/loadtester \
--namespace=test \
--set cmd.timeout=1h
```

If you need to uninstall Flagger-LoadTester, use:
```
helm -n test uninstall flagger-loadtester 
```

9) Apply the CRD:
```
./flux-flagger/crd
k apply -f ./crd-traefik-ingressroute.yaml
```

10) Apply the Webhook:
```
./flux-flagger/impl/nkp/policies
k apply -f ./network-policy-kommander-flux-allow-webhooks.yaml
```

11) Setup spring-cloud-petclinic HPA:
```
./flux-flagger/impl/nkp/spring-cloud-petclinic
k apply -f ./hpa-spring-cloud-petclinic-service.yaml
```

12) Deploy Initial Version of spring-cloud-petclinic:
```
./flux-flagger/impl/nkp/spring-cloud-petclinic
k apply -f ./deployment-spring-cloud-petclinic-service.yaml
```

Once deployment completes, you should see 1 Deployment with 2 replicas.  You can monitor the log files of the pods via K9s.  Again, it takes about 60-65 seconds for the application to startup.

13) Deploy the Canary Blue/Green:
```
./flux-flagger/impl/nkp/spring-cloud-petclinic
k apply -f ./canary-spring-cloud-petclinic-service-bg.yaml
```

Once deployed, you should see 3 services for petclinic + 1 for flagger-loadtester:
```
k get svc
```

Should show something like:
```
NAME                                     TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
flagger-loadtester                       ClusterIP   10.110.249.152   <none>        80/TCP     21h
spring-cloud-petclinic-service           ClusterIP   10.105.45.26     <none>        8080/TCP   21h
spring-cloud-petclinic-service-canary    ClusterIP   10.102.183.171   <none>        8080/TCP   21h
spring-cloud-petclinic-service-primary   ClusterIP   10.107.166.185   <none>        8080/TCP   21h
```

14) Monitor the Flagger Logs, you should see it pick up the canary for spring-cloud-petclinic:
```
k -n kommander logs deployment/flagger -f | jq .msg
```

15) Monitor K8s Events:
```
k get events
```

16) Monitor the status of the Canary, wait for it to become ready:
```
k describe canaries.flagger.app spring-cloud-petclinic-service
```

17) Spin up an instance [ pod ]of NetShoot:
```
k run netshoot -it --image=nicolaka/netshoot -- bash # Initial Access
```

18) Once netshoot is ready, exec into it:
```
k exec netshoot -it  -- bash                         # Subsequent Access
```

19) From inside netshoot, run curl to check application instances, versions

Initial:
```
curl -s http://spring-cloud-petclinic-service.test:8080/actuator/health | grep applicationVersion                       # Initial Instance
```
Canary:
```
curl -s http://spring-cloud-petclinic-service-canary.test:8080/actuator/health	| grep applicationVersion		 # Canary Instance
```
Primary:
```
curl -s http://spring-cloud-petclinic-service-primary.test:8080/actuator/health	| grep applicationVersion		 # Primary Instance
```

20) Change the version of spring-cloud-petclinic:
```
k -n test set image deployment/spring-cloud-petclinic-service \
spring-cloud-petclinic-service=ghcr.io/ericm48-gh-org/spring-cloud-petclinic-service:3.0.18-RELEASE
```

21) Monitor the Flagger Logs, you should see it pick up the canary for spring-cloud-petclinic:
```
k -n kommander logs deployment/flagger -f | jq .msg
```

This will take a couple of seconds to get started, especially for the very first time.  You should see where the Canary has detected the image-change, and start the deployment-phases [ there are 10 ].  Typically around phase 6-7 you will see the HPA's scale up the deployments due to cpu pressure, and will go to 4 replicas.
























## Clean-Up

