## Module 3. Containerization: Practical Task

# Deployment:

Currently kubernetes run in GCP (grid project). URL: http://34.102.201.195/

![[Pasted image 20260402134735.png]]

![[Pasted image 20260402134746.png]]

![[Pasted image 20260402134803.png]]


# Implementation

First I decided to start only with 1 microservice. Later I will add other ones

1) Containerize the applications from [GoogleCloudPlatform/microservices-demo](https://www.google.com/url?q=https://github.com/GoogleCloudPlatform/microservices-demo/tree/main&sa=D&source=editors&ust=1775059026590296&usg=AOvVaw3kzqpyxoCzXb8-V06s0uf_). Use the smallest possible images(scratch, alpine,  Google Distroless). In case multi-stage build is applicable then use it.
   
   Docker file for emailservice
   
   ![[Pasted image 20260401183740.png]]

Successfully build and run

![[Pasted image 20260401183818.png]]

2)  Use docker scanning tools to check whether created Dockerfiles follow best practices. Also scan the container image itself. Some examples of tools that can be used: Trivy, Checkov, Docker Scout.

Changed docker file to make it pass this check
![[Pasted image 20260401191852.png]]

And all checks passed in Checkov
![[Pasted image 20260401191836.png]]


3) Push the container images to the cloud managed container registry.

![[Pasted image 20260402054852.png]]


4. Deploy cloud managed kubernetes cluster and all required infrastructure like network. There are no specific requirements for the configuration of network, IAM, kubernetes cluster, etc. Resources can be created both via CLI, Terraform, Web Console, etc.
5. Grant kubernetes cluster permissions to pull images from managed container registry.



``` bash
06:25:17 ybryshten@noc-Latitude-5440 ~/github/microservices-demo/src/emailservice  $ gcloud container clusters create-auto microservices-cluster-v2 \
    --region=europe-west1 \
    --network=microservices-vpc \
    --subnetwork=microservices-subnet-be \
    --project=gd-gcp-gridu-devops-t1-t2
Creating cluster microservices-cluster-v2 in europe-west1... Cluster is being health-checked...⠛      
Creating cluster microservices-cluster-v2 in europe-west1... Cluster is being health-checked (Kubernetes Control Plane is healthy)...done.
Created [https://container.googleapis.com/v1/projects/gd-gcp-gridu-devops-t1-t2/zones/europe-west1/clusters/microservices-cluster-v2].
To inspect the contents of your cluster, go to: https://console.cloud.google.com/kubernetes/workload_/gcloud/europe-west1/microservices-cluster-v2?project=gd-gcp-gridu-devops-t1-t2
kubeconfig entry generated for microservices-cluster-v2.
NAME                      LOCATION      MASTER_VERSION      MASTER_IP      MACHINE_TYPE   NODE_VERSION        NUM_NODES  STATUS   STACK_TYPE
microservices-cluster-v2  europe-west1  1.35.1-gke.1396002  34.62.211.224  ek-standard-8  1.35.1-gke.1396002  3          RUNNING  IPV4

```

6. Create kubernetes manifests to deploy the application

- No credentials for redis should be provided to carts microservice. Carts service is able to use in memory store.
- Use HorizontalPodAutoscaler resource to archive autoscaling of some of the microservices.
- Use cloud solution for Ingress to expose frontend application.
- Use network policies to restrict access from other namespaces.
  
  
I start only with email service for now. First I update network policy for cluster:

``` bash
07:24:14 ybryshten@noc-Latitude-5440 ~/github/microservices-demo/src/emailservice  $ gcloud container clusters update microservices-demo-cluster \
    --zone=europe-west1-b \
    --update-addons=NetworkPolicy=ENABLED \
    --project=gd-gcp-gridu-devops-t1-t2
Updating microservices-demo-cluster...done.                                                                                                                      
Updated [https://container.googleapis.com/v1/projects/gd-gcp-gridu-devops-t1-t2/zones/europe-west1-b/clusters/microservices-demo-cluster].
To inspect the contents of your cluster, go to: https://console.cloud.google.com/kubernetes/workload_/gcloud/europe-west1-b/microservices-demo-cluster?project=gd-gcp-gridu-devops-t1-t2
07:40:41 ybryshten@noc-Latitude-5440 ~/github/microservices-demo/src/emailservice  $ gcloud container clusters update microservices-demo-cluster \
    --zone=europe-west1-b \
    --enable-network-policy \
    --project=gd-gcp-gridu-devops-t1-t2
Enabling/Disabling Network Policy causes a rolling update of all cluster nodes, similar to performing a cluster upgrade.  This operation is long-running and will 
block other operations on the cluster (including delete) until it has run to completion.

Do you want to continue (Y/n)?  y 

Updating microservices-demo-cluster...done.                                                                                                                      
Updated [https://container.googleapis.com/v1/projects/gd-gcp-gridu-devops-t1-t2/zones/europe-west1-b/clusters/microservices-demo-cluster].
To inspect the contents of your cluster, go to: https://console.cloud.google.com/kubernetes/workload_/gcloud/europe-west1-b/microservices-demo-cluster?project=gd-gcp-gridu-devops-t1-t2
07:41:45 ybryshten@noc-Latitude-5440 ~/github/microservices-demo/src/emailservice  $ 
```

Testing network policy. We can se I can access the node from defailt namespace, but not from others.

``` bash

07:55:23 ybryshten@noc-Latitude-5440 ~/github/microservices-demo/src/emailservice  $ kubectl run test-inside2 -n default --rm -i --tty --image=alpine -- sh
Warning: BCID failed open: BCID Constraint disabled by Giraffe
All commands and output from this session will be recorded in container logs, including credentials and sensitive information passed through the command prompt.
If you don't see a command prompt, try pressing enter.
/ # apk add curl
( 1/10) Installing brotli-libs (1.2.0-r0)
( 2/10) Installing c-ares (1.34.6-r0)
( 3/10) Installing libunistring (1.4.1-r0)
( 4/10) Installing libidn2 (2.3.8-r0)
( 5/10) Installing nghttp2-libs (1.68.0-r0)
( 6/10) Installing nghttp3 (1.13.1-r0)
( 7/10) Installing libpsl (0.21.5-r3)
( 8/10) Installing zstd-libs (1.5.7-r2)
( 9/10) Installing libcurl (8.17.0-r1)
(10/10) Installing curl (8.17.0-r1)
Executing busybox-1.37.0-r30.trigger
OK: 13.2 MiB in 26 packages
/ # curl -v --connect-timeout 5 http://emailservice.default.svc.cluster.local:8080
* Host emailservice.default.svc.cluster.local:8080 was resolved.
* IPv6: (none)
* IPv4: 34.118.238.210
*   Trying 34.118.238.210:8080...
* Established connection to emailservice.default.svc.cluster.local (34.118.238.210 port 8080) from 10.32.2.4 port 57262 
* using HTTP/1.x
> GET / HTTP/1.1
> Host: emailservice.default.svc.cluster.local:8080
> User-Agent: curl/8.17.0
> Accept: */*
> 
* Request completely sent off
* Received HTTP/0.9 when not allowed
* closing connection #0
curl: (1) Received HTTP/0.9 when not allowed
/ # E0402 07:56:05.422825   58120 v2.go:129] "Unhandled Error" err="next reader: read tcp 192.168.0.107:59748->34.77.126.146:443: read: connection reset by peer"
                                                                                                                                                                 E0402 07:56:05.422841   58120 v2.go:150] "Unhandled Error" err="next reader: read tcp 192.168.0.107:59748->34.77.126.146:443: read: connection reset by peer"
                                                                                                                                                            warning: couldn't attach to pod/test-inside2, falling back to streaming logs: error reading from error stream: next reader: read tcp 192.168.0.107:59748->34.77.126.146:443: read: connection reset by peer
/ # apk add curl
( 1/10) Installing brotli-libs (1.2.0-r0)
( 2/10) Installing c-ares (1.34.6-r0)
( 3/10) Installing libunistring (1.4.1-r0)
( 4/10) Installing libidn2 (2.3.8-r0)
( 5/10) Installing nghttp2-libs (1.68.0-r0)
( 6/10) Installing nghttp3 (1.13.1-r0)
( 7/10) Installing libpsl (0.21.5-r3)
( 8/10) Installing zstd-libs (1.5.7-r2)
( 9/10) Installing libcurl (8.17.0-r1)
(10/10) Installing curl (8.17.0-r1)
Executing busybox-1.37.0-r30.trigger
^[[76;5ROK: 13.2 MiB in 26 packages
/ # curl -v --connect-timeout 5 http://emailservice.default.svc.cluster.local:8080
* Host emailservice.default.svc.cluster.local:8080 was resolved.
* IPv6: (none)
* IPv4: 34.118.238.210
*   Trying 34.118.238.210:8080...
* Established connection to emailservice.default.svc.cluster.local (34.118.238.210 port 8080) from 10.32.2.4 port 57262 
* using HTTP/1.x
> GET / HTTP/1.1
> Host: emailservice.default.svc.cluster.local:8080
> User-Agent: curl/8.17.0
> Accept: */*
> 
* Request completely sent off
* Received HTTP/0.9 when not allowed
* closing connection #0
curl: (1) Received HTTP/0.9 when not allowed
^[[76;5Rpod "test-inside2" deleted from default namespace
07:56:05 ybryshten@noc-Latitude-5440 ~/github/microservices-demo/src/emailservice  $ 6;5R6;5R

```



Now I adding other 2 services to kubernetes, `cartservice` and 'frontend'

Cartservice `dockerfile`:

![[Pasted image 20260402102545.png]]

![[Pasted image 20260402102609.png]]

For `frontend` service.

![[Pasted image 20260402114102.png]]

![[Pasted image 20260402114115.png]]

And created manifests for these services:
FRONTEND:
``` yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 65532 # Using the Distroless non-root user
      containers:
      - name: server
        image: europe-central2-docker.pkg.dev/gd-gcp-gridu-devops-t1-t2/bryshten-t1-t2-repo/frontend:v1
        ports:
        - containerPort: 8080
        env:
        - name: PORT
          value: "8080"
        - name: PRODUCT_CATALOG_SERVICE_ADDR
          value: "productcatalogservice:8080"
        - name: CURRENCY_SERVICE_ADDR
          value: "currencyservice:8080"
        - name: CART_SERVICE_ADDR
          value: "cartservice:7070"
        - name: RECOMMENDATION_SERVICE_ADDR
          value: "recommendationservice:8080"
        - name: SHIPPING_SERVICE_ADDR
          value: "shippingservice:8080"
        - name: CHECKOUT_SERVICE_ADDR
          value: "checkoutservice:8080"
        - name: AD_SERVICE_ADDR
          value: "adservice:8080"
        - name: SHOPPING_ASSISTANT_SERVICE_ADDR
          value: "shoppingassistantservice:8080"
        readinessProbe:
          initialDelaySeconds: 10
          httpGet:
            path: "/_healthz"
            port: 8080
        resources:
          requests:
            cpu: 10m
            memory: 32Mi
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  type: NodePort # CRITICAL for GCP Ingress
  selector:
    app: frontend
  ports:
  - name: http
    port: 80
    targetPort: 8080
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cloud-ingress
  annotations:
    kubernetes.io/ingress.class: "gce" # This creates the GCP Load Balancer
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: frontend-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: frontend
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
```

CART SERVICE:

``` bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cartservice
spec:
  selector:
    matchLabels:
      app: cartservice
  template:
    metadata:
      labels:
        app: cartservice
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
      containers:
      - name: server
        image: europe-central2-docker.pkg.dev/gd-gcp-gridu-devops-t1-t2/bryshten-t1-t2-repo/cartservice:v1
        ports:
        - containerPort: 7070
        readinessProbe:
          initialDelaySeconds: 15
          tcpSocket:
            port: 7070
        livenessProbe:
          initialDelaySeconds: 15
          tcpSocket:
            port: 7070
        resources:
          requests:
            cpu: 10m
            memory: 32Mi
          limits:
            cpu: 100m
            memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: cartservice
spec:
  type: ClusterIP
  selector:
    app: cartservice
  ports:
  - name: grpc
    port: 7070
    targetPort: 7070
```

And everything seems work:

``` bash
12:39:11 ybryshten@noc-Latitude-5440 ~/github/microservices-demo/src/frontend  $ kubectl get all
NAME                                READY   STATUS    RESTARTS   AGE
pod/cartservice-86b44594c8-4hbsb    1/1     Running   0          6m17s
pod/emailservice-7cc559d8c7-z9xx4   1/1     Running   0          6m54s
pod/frontend-75bd767bd7-m7gx6       1/1     Running   0          5m49s

NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/cartservice    ClusterIP   34.118.231.61   <none>        7070/TCP       6m18s
service/emailservice   ClusterIP   34.118.227.80   <none>        8080/TCP       6m55s
service/frontend       NodePort    34.118.231.41   <none>        80:31520/TCP   5m50s
service/kubernetes     ClusterIP   34.118.224.1    <none>        443/TCP        5h30m

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/cartservice    1/1     1            1           6m18s
deployment.apps/emailservice   1/1     1            1           6m55s
deployment.apps/frontend       1/1     1            1           5m50s

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/cartservice-86b44594c8    1         1         1       6m18s
replicaset.apps/emailservice-7cc559d8c7   1         1         1       6m55s
replicaset.apps/frontend-75bd767bd7       1         1         1       5m50s

NAME                                                   REFERENCE                 TARGETS              MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/emailservice-hpa   Deployment/emailservice   cpu: 1%/60%          1         4         1          4h55m
horizontalpodautoscaler.autoscaling/frontend-hpa       Deployment/frontend       cpu: <unknown>/60%   1         5         0          14s
12:39:18 ybryshten@noc-Latitude-5440 ~/github/microservices-demo/src/frontend  $ 
```

![[Pasted image 20260402124052.png]]


Added other services needed for demo:

``` bash
13:43:53 ybryshten@noc-Latitude-5440 ~/github/microservices-demo/src/adservice  $ kubectl get all
NAME                                         READY   STATUS    RESTARTS   AGE
pod/adservice-768d57ddc-8ff2x                1/1     Running   0          6m34s
pod/cartservice-86b44594c8-4hbsb             1/1     Running   0          72m
pod/checkoutservice-86b6f5f965-569tj         1/1     Running   0          24m
pod/currencyservice-5cff4946c6-qn6zh         1/1     Running   0          48m
pod/emailservice-7cc559d8c7-z9xx4            1/1     Running   0          73m
pod/frontend-5cc9d84498-5bchx                1/1     Running   0          16m
pod/paymentservice-58ffdc4f7d-4g8cn          1/1     Running   0          19m
pod/productcatalogservice-578b949656-rk2zg   1/1     Running   0          45m
pod/recommendationservice-595c6796f-q5b28    1/1     Running   0          10m
pod/shippingservice-9f8ddb44-79tdb           1/1     Running   0          37m

NAME                            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
service/adservice               ClusterIP   34.118.234.149   <none>        9555/TCP       6m34s
service/cartservice             ClusterIP   34.118.231.61    <none>        7070/TCP       72m
service/checkoutservice         ClusterIP   34.118.229.167   <none>        5050/TCP       24m
service/currencyservice         ClusterIP   34.118.232.60    <none>        7000/TCP       48m
service/emailservice            ClusterIP   34.118.227.80    <none>        8080/TCP       73m
service/frontend                NodePort    34.118.231.41    <none>        80:31520/TCP   72m
service/kubernetes              ClusterIP   34.118.224.1     <none>        443/TCP        6h36m
service/paymentservice          ClusterIP   34.118.227.146   <none>        50051/TCP      19m
service/productcatalogservice   ClusterIP   34.118.234.105   <none>        3550/TCP       45m
service/recommendationservice   ClusterIP   34.118.228.25    <none>        8080/TCP       32m
service/shippingservice         ClusterIP   34.118.235.167   <none>        50051/TCP      37m

NAME                                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/adservice               1/1     1            1           6m35s
deployment.apps/cartservice             1/1     1            1           72m
deployment.apps/checkoutservice         1/1     1            1           24m
deployment.apps/currencyservice         1/1     1            1           48m
deployment.apps/emailservice            1/1     1            1           73m
deployment.apps/frontend                1/1     1            1           72m
deployment.apps/paymentservice          1/1     1            1           19m
deployment.apps/productcatalogservice   1/1     1            1           45m
deployment.apps/recommendationservice   1/1     1            1           32m
deployment.apps/shippingservice         1/1     1            1           37m
```


END!.