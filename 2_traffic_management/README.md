# Traffic Management 40%

* [Controlling network traffic flows within a service mesh]()
* [Configuring sidecar injection](#configuring-sidecar-injection)
* [Using the Gateway resource to configure ingress and egress traffic](#using-the-gateway-resource-to-configure-ingress-and-egress-traffic)
* [Understanding how to use ServiceEntry resources for adding entries to internal service registry](#understanding-how-to-use-serviceentry-resources-for-adding-entries-to-internal-service-registry)
* [Define traffic policies using DestinationRule](#define-traffic-policies-using-destinationrule)
* [Configure traffic mirroring capabilities](#configure-traffic-mirroring-capabilities)

[Docs](https://istio.io/latest/docs/concepts/traffic-management/)

[Configuration Reference](https://istio.io/latest/docs/reference/config/networking/)

[Task - Exam](https://istio.io/latest/docs/tasks/traffic-management/)

## Controlling network traffic flows within a service mesh

[Understanding Traffic Routing](https://istio.io/latest/docs/ops/configuration/traffic-management/traffic-routing/)

[Traffic Management](https://istio.io/latest/docs/concepts/traffic-management/)

### Ex1: Request Routing: Route to version 1

This task shows you how to route requests dynamically to multiple versions of a microservice.

Requirements:

* Deploy BookinInfo app.
* Route all the BookingInfo services created to the version v1.
* Check that all the traffic is going to the v1 version.

<details>
  <summary>Solution</summary>

  [Task - Request Routing](https://istio.io/latest/docs/tasks/traffic-management/request-routing/)

  ```bash
  # Get the version labels from the deployments
  $ kubectl get deploy -n default --show-labels
NAME             READY   UP-TO-DATE   AVAILABLE   AGE   LABELS
details-v1       1/1     1            1           17m   app=details,version=v1
ratings-v1       1/1     1            1           17m   app=ratings,version=v1
reviews-v3       1/1     1            1           17m   app=reviews,version=v3
reviews-v1       1/1     1            1           17m   app=reviews,version=v1
reviews-v2       1/1     1            1           17m   app=reviews,version=v2
productpage-v1   1/1     1            1           17m   app=productpage,version=v1

  # Create a DR to create the Subsets for each service/version
  $ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: bookinfo-details-route-version-1
spec:
  host: details.default.svc.cluster.local
  subsets:
  - name: v1
    labels:
      version: v1
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: bookinfo-ratings-route-version-1
spec:
  host: ratings.default.svc.cluster.local
  subsets:
  - name: v1
    labels:
      version: v1
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: bookinfo-reviews-route-version-1
spec:
  host: reviews.default.svc.cluster.local
  subsets:
  - name: v1
    labels:
      version: v1
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: bookinfo-productpage-route-version-1
spec:
  host: productpage.default.svc.cluster.local
  subsets:
  - name: v1
    labels:
      version: v1
      app: productpage
EOF
  # Create the VS to route the traffic to version 1 for all the services
  $ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: bookinfo-details-route-version-1
spec:
  hosts:
  - details.default.svc.cluster.local
  http:
  - name: "details-v1-route"
    route:
    - destination:
        host: details.default.svc.cluster.local
        port:
          number: 9080
        subset: v1
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: bookinfo-ratings-route-version-1
spec:
  hosts:
  - ratings.default.svc.cluster.local
  http:
  - name: "ratings-v1-route"
    route:
    - destination:
        host: ratings.default.svc.cluster.local
        port:
          number: 9080
        subset: v1
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: bookinfo-reviews-route-version-1
spec:
  hosts:
  - reviews.default.svc.cluster.local
  http:
  - name: "reviews-v1-route"
    route:
    - destination:
        host: reviews.default.svc.cluster.local
        port:
          number: 9080
        subset: v1
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: bookinfo-productpage-route-version-1
spec:
  hosts:
  - productpage.default.svc.cluster.local
  http:
  - name: "productpage-v1-route"
    route:
    - destination:
        host: productpage.default.svc.cluster.local
        port:
          number: 9080
        subset: v1
EOF
  ```
</details>


### Ex2: Route based on user identity

Requirements:

* Following the deployed BookinInfo app in the previous step.
* Change the route configuration so that all traffic from a specific user is routed to a specific service version.
  * All traffic from a user named Jason will be routed to the service reviews:v2.
* Check that when the user is logged in with the user Jason, the reviews service is answering with version v2.

<details>
  <summary>Solution</summary>

  ```bash

  # Update the DR to add the Subset version 2 for reviews service.
  $ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: bookinfo-reviews-route-version-1
spec:
  host: reviews.default.svc.cluster.local
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
EOF

  # Check that the reviews has 2 valid endpoints for the 2 version (v1|v2)
  $ istioctl pc endpoints deploy/productpage-v1  |grep reviews |grep "|v"
10.42.0.11:9080                                         HEALTHY     OK                outbound|9080|v2|reviews.default.svc.cluster.local
10.42.0.14:9080                                         HEALTHY     OK                outbound|9080|v1|reviews.default.svc.cluster.local

  # Update the VS for reviews to send the traffic to v2 if the user name is Jason
  $ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: bookinfo-reviews-route-version-1
spec:
  hosts:
  - reviews.default.svc.cluster.local
  http:
  - name: "reviews-v2-route"
    match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews.default.svc.cluster.local
        port:
          number: 9080
        subset: v2
  - name: "reviews-v1-route"
    route:
    - destination:
        host: reviews.default.svc.cluster.local
        port:
          number: 9080
        subset: v1
EOF

  ```
</details>

### Ex3: TCP Traffic Shifting: Weight-based TCP routing

Requirements:

* You'll need to deploy from the istio samples the sleep and tcp-echo-services.
* You need to create a dedicated TCP gateway for load balancing that TCP traffic.
* You will send 100% of the TCP traffic to tcp-echo:v1.
* After, You will route 20% of the TCP traffic to tcp-echo:v2.
* Check that it working as expected.

<details>
  <summary>Solution</summary>

  [Task - TCP Traffic Shifting](https://istio.io/latest/docs/tasks/traffic-management/tcp-traffic-shifting/)
  
  ```bash
  # We need to create a Gateway to configure our TCP routing traffic.
  # Since we installed istioctl install --set profile=demo we have already a istio ingress gateway up and running.
  $ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: tcp-echo-gateway
  namespace: istio-io-tcp-traffic-shifting
spec:
  selector:
    app: istio-ingressgateway # --set profile=demo
  servers:
  - port:
      number: 31400
      name: tcp-echo
      protocol: TCP
    hosts:
    - "*"
EOF
  # We create the 2 subset versions for the tcp-echo service
  $ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: tcp-echo
  namespace: istio-io-tcp-traffic-shifting
spec:
  host: tcp-echo
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
EOF
  # We create a TCPRoute to send the 100% traffic to the tcp-echo service v1 if the ingress traffic match on 31400.
  $ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: tcp-echo
  namespace: istio-io-tcp-traffic-shifting
spec:
  hosts:
  - tcp-echo
  gateways:
  - tcp-echo-gateway
  tcp:
  - match:
    - port: 31400
    route:
    - destination:
        host: tcp-echo
        port:
          number: 9000
        subset: v1
      weight: 100
EOF

  # Confirm that the tcp-echo service is up and running by sending some TCP traffic to the version v1.
  $ export TCP_INGRESS_PORT=$(kubectl get svc -l app=istio-ingressgateway -n istio-system -o jsonpath='{.items[0].spec.ports[?(@.name=="tcp")].port}')
  $ export INGRESS_HOST=$(kubectl -n istio-system get service -l app=istio-ingressgateway -o jsonpath='{.items[0].spec.clusterIPs[0]}')
  $ for i in {1..20}; do \
kubectl exec "$SLEEP" -c sleep -n istio-io-tcp-traffic-shifting -- sh -c "(date; sleep 1) | nc $INGRESS_HOST $TCP_INGRESS_PORT"; \
done
one Sun Jan 21 16:29:50 UTC 2024
one Sun Jan 21 16:29:51 UTC 2024
one Sun Jan 21 16:29:52 UTC 2024
one Sun Jan 21 16:29:54 UTC 2024
one Sun Jan 21 16:29:55 UTC 2024
one Sun Jan 21 16:29:56 UTC 2024

  # Transfer 20% of the traffic from tcp-echo:v1 to tcp-echo:v2:
  $ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: tcp-echo
  namespace: istio-io-tcp-traffic-shifting
spec:
  hosts:
  - tcp-echo
  gateways:
  - tcp-echo-gateway
  tcp:
  - match:
    - port: 31400
    route:
    - destination:
        host: tcp-echo
        port:
          number: 9000
        subset: v1
      weight: 80
    - destination:
        host: tcp-echo
        port:
          number: 9000
        subset: v2
      weight: 20
EOF
  # You should now notice that about 20% of the timestamps have a prefix of two, which means that 80% of the TCP traffic was routed to the v1 version of the tcp-echo service, while 20% was routed to v2.
  $ for i in {1..20}; do \
kubectl exec "$SLEEP" -c sleep -n istio-io-tcp-traffic-shifting -- sh -c "(date; sleep 1) | nc $INGRESS_HOST $TCP_INGRESS_PORT"; \
done
one Sun Jan 21 16:31:50 UTC 2024
one Sun Jan 21 16:31:51 UTC 2024
two Sun Jan 21 16:31:53 UTC 2024
one Sun Jan 21 16:31:54 UTC 2024
one Sun Jan 21 16:31:55 UTC 2024
one Sun Jan 21 16:31:56 UTC 2024
one Sun Jan 21 16:31:57 UTC 2024
two Sun Jan 21 16:31:58 UTC 2024
one Sun Jan 21 16:31:59 UTC 2024
  ```
</details>

### Ex4: Traffic Shifting: Weight-based HTTP routing

Requirements:

* Deploy BookinInfo app
* You will send 50% of traffic to reviews:v1 and 50% to reviews:v3 service.
* Then, you will complete the migration by sending 100% of traffic to reviews:v3.
* Check that is working as expected.

<details>
  <summary>Solution</summary>

  [Task - Traffic Shifting](https://istio.io/latest/docs/tasks/traffic-management/traffic-shifting/)
  
  ```bash
  # Create a DR for the reviews service to specify all the subset versions
  $ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews
  namespace: default
spec:
  host: reviews
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  - name: v3
    labels:
      version: v3
EOF

  # Create a VS to send all the traffic to the v1 version
  $ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews-route
  namespace: default
spec:
  hosts:
  - reviews
  http:
  - name: "reviews-v1-route"
    route:
    - destination:
        host: reviews
        subset: v1
EOF
  # Transfer 50% of the traffic from reviews:v1 to reviews:v3
  $ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews-route
  namespace: default
spec:
  hosts:
  - reviews
  http:
  - name: "reviews-v1-v3-route"
    route:
    - destination:
        host: reviews
        subset: v1
      weight: 50
    - destination:
        host: reviews
        subset: v3
      weight: 50
EOF
  # Refresh the /productpage in your browser and you now see red colored star ratings approximately 50% of the time. This is because the v3 version of reviews accesses the star ratings service, but the v1 version does not.

  # Route 100% of the traffic to reviews:v3 by applying this virtual service:
  $ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews-route
  namespace: default
spec:
  hosts:
  - reviews
  http:
  - name: "reviews-v1-v3-route"
    route:
    - destination:
        host: reviews
        subset: v1
      weight: 0
    - destination:
        host: reviews
        subset: v3
      weight: 100
EOF
  ```
</details>

## Configuring sidecar injection

[Setup - Installing the Sidecar](https://istio.io/latest/docs/setup/additional-setup/sidecar-injection/)

### Ex5:Automatic sidecar injection

#### Ex5.1: Inject httpbin service automatically at namespace level.

Requirements:

* Inject httpbin service automatically at namespace level.
* Check that httpbin is automatically injected when it's deployed.

<details>
  <summary>Solution</summary>

  [Setup - Installing the Sidecar](https://istio.io/latest/docs/setup/additional-setup/sidecar-injection)
  
  ```bash
  # Label the default namespace with istio-injection=enabled
  $ kubectl label namespace default istio-injection=enabled --overwrite
  # Deploy httpbin service to injected automatically
  $ kubectl apply -f samples/httpbin/httpbin.yaml
  # Get the httpbin pods, we should see 2 READY containers where istio-proxy will be running with the main application.
  $ kubectl get po -l app=httpbin
NAME                       READY   STATUS    RESTARTS   AGE
httpbin-65975d4c6f-9v5kr   2/2     Running   0          59s
  ```
</details>

#### Ex5.2: Inject httpbin service automatically at workload level. Only httpbin service should be automatically injected inside the default namespace.

Requirements:

* Inject httpbin service automatically at workload level. 
  * Only httpbin service should be automatically injected inside the default namespace.
* Check that httpbin service is ONLY automatically injected when it's deployed.

<details>
  <summary>Solution</summary>

  [Setup - Installing the Sidecar](https://istio.io/latest/docs/setup/additional-setup/sidecar-injection)
  
  ```bash
  # Disable injection for the default namespace
  $ kubectl label namespace default istio-injection-
  # Remove httpbin service
  $ kubectl delete -f samples/httpbin/httpbin.yaml
  # Modify httpbin deployment yaml to add the Pod label injection
  $ kubectl apply -f - <<EOF
##################################################################################################
# httpbin service
##################################################################################################
apiVersion: v1
kind: ServiceAccount
metadata:
  name: httpbin
---
apiVersion: v1
kind: Service
metadata:
  name: httpbin
  labels:
    app: httpbin
    service: httpbin
spec:
  ports:
  - name: http
    port: 8000
    targetPort: 80
  selector:
    app: httpbin
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpbin
spec:
  replicas: 1
  selector:
    matchLabels:
      app: httpbin
      version: v1
  template:
    metadata:
      labels:
        app: httpbin
        version: v1
        sidecar.istio.io/inject: "true" # Istio Pod label injection
    spec:
      serviceAccountName: httpbin
      containers:
      - image: docker.io/kong/httpbin
        imagePullPolicy: IfNotPresent
        name: httpbin
        ports:
        - containerPort: 80
EOF
  # Get the httpbin pods, we should see 2 READY containers where istio-proxy will be running with the main application.
  $ kubectl get po -l app=httpbin
NAME                       READY   STATUS    RESTARTS   AGE
httpbin-65975d4c6f-9v5kr   2/2     Running   0          59s
  ```

</details>

### Manual sidecar injection

#### Ex5.3: Inject httpbin service manually.

Requirements:

* Inject httpbin service manually.
* Check that httpbin service is automatically injected.

<details>
  <summary>Solution</summary>

  #### istioctl kube-inject --help (Check - Examples:)

  [Setup - Manual sidecar injection](https://istio.io/latest/docs/setup/additional-setup/sidecar-injection/#manual-sidecar-injection)
  
  ```bash
  # Manual sidecar injection using cluster configurations
  $ istioctl kube-inject -f  ./samples/httpbin/httpbin.yaml | kubectl apply -f -
  ```
</details>

#### Ex5.4: Inject httpbin service manually persisting the injection file.

Requirements:

* Inject httpbin service manually persisting the injection file.
* Check that httpbin service is injected.

<details>
  <summary>Solution</summary>

  #### istioctl kube-inject --help (Check - Examples:)

  [Setup -Installing the Sidecar](https://istio.io/latest/docs/setup/additional-setup/sidecar-injection)
  
  ```bash
  # Create a persistent version of the deployment with Istio sidecar injected.
  $ istioctl kube-inject -f  ./samples/httpbin/httpbin.yaml -o httpbin-injected.yaml
  ```
</details>


#### Ex5.5: Inject httpbin service already running manually

Requirements:

* Inject httpbin service already running manually.
* Check that httpbin service is injected.

<details>
  <summary>Solution</summary>

  #### istioctl kube-inject --help (Check - Examples:)

  [Setup -Installing the Sidecar](https://istio.io/latest/docs/setup/additional-setup/sidecar-injection)
  
  ```bash
  $ kubectl get po -n default
NAME                       READY   STATUS    RESTARTS   AGE
httpbin-6fcb98998c-h9ss8   1/1     Running   0          21s
  # Update an existing deployment.
  $ kubectl get deploy -n default -oyaml | ./istio-1.18.2/bin/istioctl kube-inject -f - |kubectl apply -f -
deployment.apps/httpbin configured
  $ kubectl get po -n default
NAME                      READY   STATUS    RESTARTS   AGE
httpbin-5fd75cf66-7hcnp   2/2     Running   0          14s
  ```
</details>

#### Ex5.6: Inject httpbin service manually using local configuration files.

Requirements:

* Inject httpbin service manually using local configuration files.
* Check that httpbin service is injected.

<details>
  <summary>Solution</summary>

  #### istioctl kube-inject --help (Check - Examples:)

  [Setup -Installing the Sidecar](https://istio.io/latest/docs/setup/additional-setup/sidecar-injection)
  
  ```bash
  # Capture cluster configuration for later use with kube-inject
  kubectl -n istio-system get cm istio-sidecar-injector  -o jsonpath="{.data.config}" > /tmp/inj-template.tmpl
  kubectl -n istio-system get cm istio -o jsonpath="{.data.mesh}" > /tmp/mesh.yaml
  kubectl -n istio-system get cm istio-sidecar-injector -o jsonpath="{.data.values}" > /tmp/values.json

  # Use kube-inject based on captured configuration
  $ istioctl kube-inject -f ./istio-1.18.2/samples/httpbin/httpbin.yaml \
    --injectConfigFile /tmp/inj-template.tmpl \
    --meshConfigFile /tmp/mesh.yaml \
    --valuesFile /tmp/values.json \
| kubectl apply -f -
  ```
</details>

## Ex6: Using the Gateway resource to configure ingress and egress traffic

[Docs Gateway](https://istio.io/latest/docs/reference/config/networking/gateway/)

#### Ex6.1 Gateway: Ingress - Create an Ingress Gateway and expose the routes: /status and /delay from the httpbin service.

[Task - Ingress](https://istio.io/latest/docs/tasks/traffic-management/ingress/)

Requirements:

* Create an Ingress Gateway
* Expose the routes: /status and /delay from the httpbin service
* Check those routes are working as expected on that Gateway.

<details>
  <summary>Solution</summary>
  
  ```bash
  # Create a Istio ingress gateway envoy service
  # Minimal profile include the Ingress and the Egress gateway.
  $ istioctl install --set profile=minimal

  # Enable automatic ns default Isito injection
  $ kubectl label ns default istio-injection=enabled

  # Create the Istio Gateway to configure the target envoy ingress.
  $ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: httpbin-gateway
  namespace: default # It should be in the same ns of the Istio Ingress?. No
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - httpbin.example.com
EOF

  # Create the Virtual Service to expose the routes /status and /delay from the httpbin service.
  $ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: httpbin
  namespace: default
spec:
  hosts:
  - httpbin.example.com
  gateways:
  - httpbin-gateway
  http:
  - match:
    - uri:
        prefix: /status
    - uri:
        prefix: /delay
    route:
    - destination:
        port:
          number: 8000 # can be omitted if it's the only port for reviews
        host: httpbin.default.svc.cluster.local
EOF

  # Access the httpbin service through the Istio ingress
  $ kubectl port-forward service/istio-ingressgateway 8080:80 -n istio-system

  $ curl -vvv -H 'Host: httpbin.example.com' localhost:8080/status/200
*   Trying [::1]:8080...
* Connected to localhost (::1) port 8080
> GET /status/200 HTTP/1.1
> Host: httpbin.example.com
> User-Agent: curl/8.4.0
> Accept: */*
>
< HTTP/1.1 200 OK
< server: istio-envoy
< date: Fri, 26 Jan 2024 17:53:34 GMT
< content-type: text/html; charset=utf-8
< access-control-allow-origin: *
< access-control-allow-credentials: true
< content-length: 0
< x-envoy-upstream-service-time: 2
<
  ```
</details>

#### Ex6.2 Gateway: Egress - Call edition.cnn.com through a dedicated Istio Egress

[Task - Egress](https://istio.io/latest/docs/tasks/traffic-management/egress/egress-gateway/)

Configure Istio to allow access to external HTTP and HTTPS services from applications inside the mesh.

Requirements:
* Create a dedicate Istio Egress gateway.
  * Host: edition.cnn.com.
* The Egress Gateway name: edition-egressgateway.
* Check that the call to edition.cnn.com goes trough that egress gateway.

<details>
  <summary>Solution</summary>
  
  ```bash
  # Install Istio and customize the IstioOperator CRD to set the proper Egress naming.
cat <<EOF >>istio-edition-egressgateway.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  profile: minimal
  components:
    egressGateways:
    - name: edition-egressgateway
      enabled: true
EOF
  $ istioctl install -f istio-edition-egressgateway.yaml
This will install the Istio 1.18.2 minimal profile with ["Istio core" "Istiod" "Egress gateways"] components into the cluster. Proceed? (y/N)

  # Label the default ns to autoinject the pods.
  $ kubectl label ns default istio-injection=enabled
  # Deploy Sleep app to use it as a client to perform the request
  $ kubectl apply -f samples/sleep/sleep.yaml

  # Enable access logs on the proxies
cat <<EOF >>istio-edition-egressgateway.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  profile: minimal
  meshConfig:
    accessLogFile: /dev/stdout # It's enable access logs.
  components:
    egressGateways:
    - name: edition-egressgateway
      enabled: true
EOF
  $ istioctl upgrade -f istio-edition-egressgateway.yaml

  # Create the Gateway CRD for the Egress
  $ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: edition-egressgateway
  namespace: default
spec:
  selector:
    app: istio-egressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - edition.cnn.com
    tls:
      httpsRedirect: true # sends 301 redirect for http requests
  - port:
      number: 443
      name: https-443
      protocol: HTTPS
    hosts:
    - edition.cnn.com
    tls:
      # https://istio.io/latest/docs/reference/config/networking/gateway/#ServerTLSSettings-TLSmode
      mode: PASSTHROUGH # The SNI string presented by the client will be used as the match criterion in a VirtualService TLS route to determine the destination service from the service registry.
EOF

  # Let's take a look to the logs from the Sleep istio-proxy container
  $ kubectl logs -f -n default -l app=sleep -c istio-proxy

  # Let's perform a request to the external host to validate that it works on a different terminal to see the logs.
  $ kubectl -n default exec $(kubectl get pod -n default -l app=sleep -ojsonpath='{.items[0].metadata.name}') -- curl -vvv -L -I http://edition.cnn.com # 200
  # Logs:
  [2024-01-27T08:16:24.063Z] "HEAD / HTTP/1.1" 301 - via_upstream - "-" 0 0 31 31 "-" "curl/8.5.0" "20e93f6d-bb96-4ffe-834a-15d44f85ea16" "edition.cnn.com" "151.101.67.5:80" PassthroughCluster 10.42.0.7:46822 151.101.67.5:80 10.42.0.7:46806 - allow_any
  [2024-01-27T08:16:24.096Z] "- - -" 0 - - - "-" 789 6074 53 - "-" "-" "-" "-" "151.101.131.5:443" PassthroughCluster 10.42.0.7:48158 151.101.131.5:443 10.42.0.7:48150 - -
  # We can see on the output of the proxy: PassthroughCluster meaning that host is not part of the mesh since it didn't match any Envoy configuration.

  # Create a ServiceEntry to add "edition.cnn.com" as part of the mesh.
  # It allow us to apply Istio features on that external host.
  # It'll create a Envoy Cluster and the Endpoints associated with it using DNS resolution.
  $ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: external-svc-edition
  namespace : default
spec:
  hosts:
  - edition.cnn.com
  location: MESH_EXTERNAL
  ports:
  - number: 80
    name: http
    protocol: HTTP
  - number: 443
    name: https
    protocol: HTTPS
  resolution: DNS
EOF

  # Check how this service entry is translated to Envoy configuration
  $ istioctl pc cluster $(kubectl get pod -n default -l app=sleep -ojsonpath='{.items[0].metadata.name}') --fqdn edition.cnn.com
  $ istioctl pc endpoint $(kubectl get pod -n default -l app=sleep -ojsonpath='{.items[0].metadata.name}')  --cluster "outbound|80||edition.cnn.com"
  $ istioctl pc endpoint $(kubectl get pod -n default -l app=sleep -ojsonpath='{.items[0].metadata.name}')  --cluster "outbound|443||edition.cnn.com"

  # Now perform the same request and pay attention to the log output
  $ kubectl -n default exec $(kubectl get pod -n default -l app=sleep -ojsonpath='{.items[0].metadata.name}') -- curl -vvv -L -I http://edition.cnn.com # 200
  # Logs:
  [2024-01-27T08:21:27.879Z] "HEAD / HTTP/1.1" 301 - via_upstream - "-" 0 0 31 31 "-" "curl/8.5.0" "dd06627c-a164-4658-a8e8-9682df451bc8" "edition.cnn.com" "151.101.195.5:80" outbound|80||edition.cnn.com 10.42.0.7:55516 151.101.195.5:80 10.42.0.7:55506 - default
  [2024-01-27T08:21:27.914Z] "- - -" 0 - - - "-" 789 6073 54 - "-" "-" "-" "-" "151.101.131.5:443" outbound|443||edition.cnn.com 10.42.0.7:56818 151.101.3.5:443 10.42.0.7:41370 edition.cnn.com -
  # We can see now that is not anymore present PassthroughCluster since we added that external host to be part of the mesh and the request match with a specific Envoy configuration.

  # Create a dedicated DR to send the request for the host edition.cnn.com trough it
  $ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: egressgateway-for-cnn
spec:
  host: edition-egressgateway.istio-system.svc.cluster.local
  subsets:
  - name: cnn
EOF

  # Create the VS to route the traffic to the Egress gateway
  $ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: external-svc-edition
spec:
  hosts:
  - "edition.cnn.com"
  gateways:
  - edition-egressgateway
  - mesh
  http:
  - name: "http-mesh-traffic"
    match:
    - gateways:
      - mesh
    route:
    - destination:
        host: edition-egressgateway.istio-system.svc.cluster.local
        subset: cnn
  - name: "http-gateway-traffic"
    match:
    - gateways:
      - edition-egressgateway
    route:
    - destination:
        host: edition.cnn.com
        port:
          number: 80
  tls:
  - match:
    - gateways:
      - mesh
      sniHosts: 
      - "edition.cnn.com"
    route:
    - destination:
        host: edition-egressgateway.istio-system.svc.cluster.local
        subset: cnn
        port:
          number: 443
  - match:
    - gateways:
      - edition-egressgateway
      sniHosts: 
      - "edition.cnn.com"
    route:
    - destination:
        host: edition.cnn.com
        port:
          number: 443
EOF

  # Let's perform a request to the external host to validate that now the traffic is passing through the Egress gateway instead of the istio-proxy from the app sidecar.
  $ kubectl -n default exec $(kubectl get pod -n default -l app=sleep -ojsonpath='{.items[0].metadata.name}') -- curl -vvv -L -I http://edition.cnn.com # 200
  $ kubectl logs -f -n default -l app=sleep -c istio-proxy
  [2024-01-27T09:24:20.239Z] "HEAD / HTTP/1.1" 301 - via_upstream - "-" 0 0 1 1 "-" "curl/8.5.0" "88c8e6a5-ff0f-4d45-adf7-a24dba560bf1" "edition.cnn.com" "10.42.0.6:8080" outbound|80||edition-egressgateway.istio-system.svc.cluster.local 10.42.0.7:33586 151.101.3.5:80 10.42.0.7:50848 - http-mesh-traffic
  [2024-01-27T09:24:20.242Z] "- - -" 0 - - - "-" 789 6073 58 - "-" "-" "-" "-" "10.42.0.6:8443" outbound|443||edition-egressgateway.istio-system.svc.cluster.local 10.42.0.7:35144 151.101.131.5:443 10.42.0.7:54822 edition.cnn.com -
  # we can see that from the istio-proxy point of view the request goes to the endpoint outbound|80||edition-egressgateway.istio-system.svc.cluster.local and then the 301 redirect from the 80 to the 443 goes to outbound|443||edition-egressgateway.istio-system.svc.cluster.local

  # Now if we chech the logs from the edition-egressgateway we should see the traffic passing through
  # We see first the 80 http traffic that it cause the 301 and then the 443 going to the outbound|443||edition.cnn.com Envoy endpoint
  $ kubectl logs -f -n istio-system -l app=istio-egressgateway 
  [2024-01-27T09:24:20.239Z] "HEAD / HTTP/2" 301 - direct_response - "-" 0 0 0 - "10.42.0.7" "curl/8.5.0" "88c8e6a5-ff0f-4d45-adf7-a24dba560bf1" "edition.cnn.com" "-" - - 10.42.0.6:8080 10.42.0.7:33586 - -
  [2024-01-27T09:24:20.243Z] "- - -" 0 - - - "-" 789 6073 56 - "-" "-" "-" "-" "151.101.195.5:443" outbound|443||edition.cnn.com 10.42.0.6:42880 10.42.0.6:8443 10.42.0.7:35144 edition.cnn.com -
  ```
</details>

## Ex7: Understanding how to use ServiceEntry resources for adding entries to internal service registry

[Docs](https://istio.io/latest/docs/reference/config/networking/service-entry/)

### Ex7.1: Add api.dropboxapi.com to the mesh [MESH_EXTERNAL]

Requirements: 
* Add api.dropboxapi.com to the mesh as [MESH_EXTERNAL]
* Validate api.dropboxapi.com: is part of the mesh and it's working as expected.

<details>
  <summary>Solution</summary>
  
  ```bash
  # Label default ns for automaticaly inject Istio sidecar
  $ kubectl label ns default istio-injection=enabled
  # Deploy sleep on the default ns
  $ kubectl apply -f samples/sleep/sleep.yaml
  # Enable access logs for the proxies
  $ cat <<EOF >>istio-logs.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  profile: minimal
  meshConfig:
    accessLogFile: /dev/stdout # It's enable access logs.
EOF
  # Upgrade Istio with that changes
  $ istioctl upgrade -f istio-logs.yaml

  # Let's perform a request from sleep service to the api.dropboxapi.com and check the logs.
  $ kubectl -n default exec $(kubectl get pod -n default -l app=sleep -ojsonpath='{.items[0].metadata.name}') -- curl -vvv -L -I http://api.dropboxapi.com
  [2024-01-27T10:39:54.925Z] "HEAD / HTTP/1.1" 301 - via_upstream - "-" 0 0 31 30 "-" "curl/8.5.0" "aa2f8365-4721-4094-a070-1de2a14c4f7f" "api.dropboxapi.com" "162.125.68.19:80" PassthroughCluster 10.42.0.7:36682 162.125.68.19:80 10.42.0.7:36676 - allow_any
  [2024-01-27T10:39:54.959Z] "- - -" 0 - - - "-" 808 4758 345 - "-" "-" "-" "-" "162.125.68.19:443" PassthroughCluster 10.42.0.7:44654 162.125.68.19:443 10.42.0.7:44638 - -
  [2024-01-27T10:39:55.240Z] "- - -" 0 - - - "-" 816 3700 65 - "-" "-" "-" "-" "162.125.68.18:443" PassthroughCluster 10.42.0.7:47158 162.125.68.18:443 10.42.0.7:47150 - -
  # We can see on the output of the proxy: PassthroughCluster meaning that host is not part of the mesh since it didn't match any Envoy configuration.

  # Let's create a SE with location: MESH_EXTERNAL since it's an external API
  $ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: external-svc-dropboxapi
  namespace : default
spec:
  hosts:
  - api.dropboxapi.com
  location: MESH_EXTERNAL
  ports:
  - number: 80
    name: http
    protocol: HTTP
  - number: 443
    name: https
    protocol: HTTPS
  resolution: DNS
EOF

  # Let's perform again the same request
  $ kubectl -n default exec $(kubectl get pod -n default -l app=sleep -ojsonpath='{.items[0].metadata.name}') -- curl -vvv -L -I http://api.dropboxapi.com
  [2024-01-27T10:43:10.476Z] "HEAD / HTTP/1.1" 301 - via_upstream - "-" 0 0 30 30 "-" "curl/8.5.0" "c6931cd0-088c-45c3-bbcc-e4e7a2f4e850" "api.dropboxapi.com" "162.125.68.19:80" outbound|80||api.dropboxapi.com 10.42.0.7:40328 162.125.68.19:80 10.42.0.7:40314 - default
  [2024-01-27T10:43:10.509Z] "- - -" 0 - - - "-" 808 4759 358 - "-" "-" "-" "-" "162.125.68.19:443" outbound|443||api.dropboxapi.com 10.42.0.7:42858 162.125.68.19:443 10.42.0.7:42856 api.dropboxapi.com -
  [2024-01-27T10:43:10.804Z] "- - -" 0 - - - "-" 816 3679 64 - "-" "-" "-" "-" "162.125.68.18:443" PassthroughCluster 10.42.0.7:42416 162.125.68.18:443 10.42.0.7:42406 www.dropbox.com -
  # Now we can see that to reach api.dropboxapi.com we're using the outbound|443||api.dropboxapi.com Envoy cluster.
  # We can see also that since it's redirecting the request we have www.dropbox.com that still PassthroughCluster since it's not part of the mesh yet, let's fix it.
  $ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: external-svc-dropboxapi
  namespace : default
spec:
  hosts:
  - api.dropboxapi.com
  - www.dropbox.com
  location: MESH_EXTERNAL
  ports:
  - number: 80
    name: http
    protocol: HTTP
  - number: 443
    name: https
    protocol: HTTPS
  resolution: DNS
EOF
  # Now we can see that all the host are part of the mesh
  $ kubectl -n default exec $(kubectl get pod -n default -l app=sleep -ojsonpath='{.items[0].metadata.name}') -- curl -vvv -L -I http://api.dropboxapi.com
  [2024-01-27T10:46:43.420Z] "HEAD / HTTP/1.1" 301 - via_upstream - "-" 0 0 31 30 "-" "curl/8.5.0" "b0ea2bb5-6a0f-44fd-bdf0-bb7370376b10" "api.dropboxapi.com" "162.125.68.19:80" outbound|80||api.dropboxapi.com 10.42.0.7:34476 162.125.68.19:80 10.42.0.7:34460 - default
  [2024-01-27T10:46:43.453Z] "- - -" 0 - - - "-" 808 4759 320 - "-" "-" "-" "-" "162.125.68.19:443" outbound|443||api.dropboxapi.com 10.42.0.7:42258 162.125.68.19:443 10.42.0.7:42248 api.dropboxapi.com -
  [2024-01-27T10:46:43.709Z] "- - -" 0 - - - "-" 816 3700 64 - "-" "-" "-" "-" "162.125.68.18:443" outbound|443||www.dropbox.com 10.42.0.7:48704 162.125.68.18:443 10.42.0.7:48692 www.dropbox.com -

  # Let's check the Envoy confiuration generated from that SE.
  $ istioctl pc endpoint $(kubectl get pod -n default -l app=sleep -ojsonpath='{.items[0].metadata.name}') |grep dropbox
162.125.68.18:80                                        HEALTHY     OK                outbound|80||www.dropbox.com
162.125.68.18:443                                       HEALTHY     OK                outbound|443||www.dropbox.com
162.125.68.19:80                                        HEALTHY     OK                outbound|80||api.dropboxapi.com
162.125.68.19:443                                       HEALTHY     OK                outbound|443||api.dropboxapi.com
  $ istioctl pc cluster $(kubectl get pod -n default -l app=sleep -ojsonpath='{.items[0].metadata.name}') |grep dropbox
api.dropboxapi.com                               80        -          outbound      STRICT_DNS
api.dropboxapi.com                               443       -          outbound      STRICT_DNS
www.dropbox.com                                  80        -          outbound      STRICT_DNS
www.dropbox.com                                  443       -          outbound      STRICT_DNS

# Istio maintains an internal service registry which can be observed through a debug endpoint /debug/registryz exposed by istiod
$ kubectl exec -n istio-system deploy/istiod -- curl -s localhost:15014/debug/registryz
  ```
</details>

### Ex7.2: Add mymongodb.somedomain to the mesh as [MESH_INTERNAL]

Requirements: 
* Add mymongodb.somedomain to the mesh as [MESH_INTERNAL]
* Validate mymongodb.somedomain: is part of the mesh and it's working as expected.

<details>
  <summary>Solution</summary>
  
  ```bash
# Create the SE to add the Mongodb as part of the mesh as location: MESH_INTERNAL
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: internal-svc-mongocluster
spec:
  hosts:
  - mymongodb.somedomain
  ports:
  - number: 27018
    name: mongodb
    protocol: MONGO
  location: MESH_INTERNAL
  resolution: STATIC
  endpoints:
  - address: 10.43.0.1
EOF

# Check the new MongoDB instances are part of the registry as internal services "MeshExternal": false
$ kubectl exec -n istio-system deploy/istiod -- \
  curl -s localhost:15014/debug/registryz | jq '.[] | select( .hostname | contains("mymongodb.somedomain"))'

  # Enable DNS Proxying to allow Istio proxy resolve the hostname from the SE
  $ cat <<EOF >>istio-logs-dns.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  profile: minimal
  meshConfig:
    accessLogFile: /dev/stdout # It's enable access logs.
    defaultConfig:
      proxyMetadata:
        # Enable basic DNS proxying
        ISTIO_META_DNS_CAPTURE: "true"
        # Enable automatic address allocation, optional
        ISTIO_META_DNS_AUTO_ALLOCATE: "true"
EOF
  $ istioctl upgrade -f istio-logs-dns.yaml
  # Check the MongoDB SE service resolve the fake domain
  $ kubectl -n default exec $(kubectl get pod -n default -l app=sleep -ojsonpath='{.items[0].metadata.name}') -- nc -v mymongodb.somedomain 27018
  mymongodb.somedomain (240.240.40.99:27018) open
  # Check the logs from the sleep istio-proxy
  # You can see that the response flag in the logs:
  # UF: UpstreamConnectionFailure
  # URX: UpstreamRetryLimitExceeded
  # It's expected since we don't have a real mongodb listening on that port.
  [2024-01-27T14:15:24.881Z] "- - -" 0 UF,URX - - "-" 0 0 10000 - "-" "-" "-" "-" "10.43.0.1:27018" outbound|27018||mymongodb.somedomain - 240.240.40.99:27018 10.42.0.10:34145 - -

  ```
</details>

## Ex8: Define traffic policies using DestinationRule

[Docs](https://istio.io/latest/docs/reference/config/networking/destination-rule/)

### Ex8.1: Change the Load balancing mechanissim on the helloworld service to : RANDOM

Requirements: 
* Change the Load balancing mechanissim on the helloworld service to : RANDOM.
* Validate that the new load balancing mechanissim is working as expected.

[Load balancing](https://tetratelabs.github.io/istio-0to60/discovery/#load-balancing)

<details>
  <summary>Solution</summary>
  
  ```bash
  # Deploy a client http service: sleep app
  $ kubectl label ns default istio-injection=enabled
  $ kubectl apply -f istio-1.18.2/samples/sleep/sleep.yaml
  # We deploy the server app: Helloworld
  $ kubectl apply -f istio-1.18.2/samples/helloworld/helloworld.yaml
  # Let's check the default load balancing mechanissim.
  $ istioctl pc cluster $(kubectl get pod -n default -l app=helloworld -ojsonpath='{.items[0].metadata.name}') --fqdn "helloworld.default.svc.cluster.local" -ojson |grep lbPolicy
  "lbPolicy": "LEAST_REQUEST",
  # Let's make some test to validate that behaviour
  $ for i in {1..6}; do kubectl exec deploy/sleep -- curl -s helloworld:5000/hello; done
    Hello version: v1, instance: helloworld-v1-867747c89-6sqrw
    Hello version: v1, instance: helloworld-v1-867747c89-6sqrw
    Hello version: v1, instance: helloworld-v1-867747c89-6sqrw
    Hello version: v1, instance: helloworld-v1-867747c89-6sqrw
    Hello version: v1, instance: helloworld-v1-867747c89-6sqrw
    Hello version: v1, instance: helloworld-v1-867747c89-6sqrw
  # Now let's change the load balancing mechanissim to RANDOM
  # Create a DR for helloworld
  $ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: helloworld
spec:
  host: helloworld.default.svc.cluster.local
  trafficPolicy:
    loadBalancer:
      simple: RANDOM
EOF
  # Let's check again if the value has been applied properly.
  $ istioctl pc cluster $(kubectl get pod -n default -l app=helloworld -ojsonpath='{.items[0].metadata.name}') --fqdn "helloworld.default.svc.cluster.local" -ojson |grep lbPolicy
        "lbPolicy": "RANDOM",
  # Let's test the new load balancing mechanissim
  $ for i in {1..6}; do kubectl exec deploy/sleep -- curl -s helloworld:5000/hello; done
    Hello version: v1, instance: helloworld-v1-867747c89-6sqrw
    Hello version: v1, instance: helloworld-v1-867747c89-6sqrw
    Hello version: v1, instance: helloworld-v1-867747c89-6sqrw
    Hello version: v1, instance: helloworld-v1-867747c89-6sqrw
    Hello version: v2, instance: helloworld-v2-7f46498c69-gm5wn
    Hello version: v2, instance: helloworld-v2-7f46498c69-gm5wn
  # Now we can see that we have random pods version returned in the response.
  ```
</details>

### Ex8.2: Traffic distribution spread proportially 50%/50% for helloworld service between v1 & v2 version

Requirements: 
* Spread proportially 50%/50% for helloworld service between v1 & v2 version.
* Validate that the traffic is spread proportially 50%/50% for helloworld service between v1 & v2 version.

[Traffic distribution](https://tetratelabs.github.io/istio-0to60/discovery/#traffic-distribution)

<details>
  <summary>Solution</summary>

```bash
  # Deploy a client http service: sleep app
  $ kubectl label ns default istio-injection=enabled
  $ kubectl apply -f istio-1.18.2/samples/sleep/sleep.yaml
  # We deploy the server app: Helloworld
  $ kubectl apply -f istio-1.18.2/samples/helloworld/helloworld.yaml
  # Create a DR to add the subsets for the different versions of helloworld.
  # That basically will create new cluster and endpoints on Envoy for that service.
  $ istioctl pc endpoint $(kubectl get pod -n default -l app=helloworld -ojsonpath='{.items[0].metadata.name}') |grep helloworld
  10.42.0.7:5000                                          HEALTHY     OK                outbound|5000||helloworld.default.svc.cluster.local
  10.42.0.8:5000                                          HEALTHY     OK                outbound|5000||helloworld.default.svc.cluster.local
  $ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: helloworld
spec:
  host: helloworld.default.svc.cluster.local
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
EOF
  # Check the new clusters and endpoints created for each specific app version
  $ istioctl pc endpoint $(kubectl get pod -n default -l app=helloworld -ojsonpath='{.items[0].metadata.name}') |grep helloworld
    10.42.0.7:5000                                          HEALTHY     OK                outbound|5000|v1|helloworld.default.svc.cluster.local
    10.42.0.7:5000                                          HEALTHY     OK                outbound|5000||helloworld.default.svc.cluster.local
    10.42.0.8:5000                                          HEALTHY     OK                outbound|5000|v2|helloworld.default.svc.cluster.local
    10.42.0.8:5000                                          HEALTHY     OK                outbound|5000||helloworld.default.svc.cluster.local
  $ istioctl pc cluster $(kubectl get pod -n default -l app=helloworld -ojsonpath='{.items[0].metadata.name}') --fqdn "helloworld.default.svc.cluster.local"
    SERVICE FQDN                             PORT     SUBSET     DIRECTION     TYPE     DESTINATION RULE
    helloworld.default.svc.cluster.local     5000     -          outbound      EDS      helloworld.default
    helloworld.default.svc.cluster.local     5000     v1         outbound      EDS      helloworld.default
    helloworld.default.svc.cluster.local     5000     v2         outbound      EDS      helloworld.default

  # Create a VS to split the traffic proportially betwenn both versions
  $ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: helloworld-route
spec:
  hosts:
  - helloworld.default.svc.cluster.local
  http:
  - name: "helloworld-routes"
    route:
    - destination:
        host: helloworld.default.svc.cluster.local
        subset: v2
      weight: 50
    - destination:
        host: helloworld.default.svc.cluster.local
        subset: v1
      weight: 50
EOF

  # Let's test if the traffic is well spread between v1 and v2
  $ for i in {1..6}; do kubectl exec deploy/sleep -- curl -s helloworld:5000/hello; done
    Hello version: v2, instance: helloworld-v2-7f46498c69-gm5wn
    Hello version: v1, instance: helloworld-v1-867747c89-6sqrw
    Hello version: v2, instance: helloworld-v2-7f46498c69-gm5wn
    Hello version: v1, instance: helloworld-v1-867747c89-6sqrw
    Hello version: v2, instance: helloworld-v2-7f46498c69-gm5wn
    Hello version: v2, instance: helloworld-v2-7f46498c69-gm5wn

  # istioctl CLI provides a convenient command to inspect the configuration of a service:
  $ istioctl x describe svc $(kubectl get svc -n default -l app=helloworld -ojsonpath='{.items[0].metadata.name}')
  Service: helloworld
    Port: http 5000/HTTP targets pod port 5000
  DestinationRule: helloworld for "helloworld.default.svc.cluster.local"
    Matching subsets: v1,v2
    No Traffic Policy
  VirtualService: helloworld-route
    Route to host "helloworld.default.svc.cluster.local" subset "v2" with weight 50%
    Route to host "helloworld.default.svc.cluster.local" subset "v1" with weight 50%
  Skipping Gateway information (no ingress gateway pods)
  ```
</details>


## Ex9: Configure traffic mirroring capabilities for helloworld service

[Docs](https://istio.io/latest/docs/tasks/traffic-management/mirroring/)

Requirements: 
* Configure traffic mirroring capabilities for helloworld service.
* Send a proportion of mirror traffic to v2 of helloworld service.

<details>
  <summary>Solution</summary>

```bash
  # Enable access logs for the proxies
  $ cat <<EOF >>istio-logs.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  profile: minimal
  meshConfig:
    accessLogFile: /dev/stdout
EOF
  $ istioctl upgrade -f istio-logs.yaml
  # Let's create a DR to set the subsets of helloworld
  $ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: helloworld
spec:
  host: helloworld.default.svc.cluster.local
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
EOF
  # Let's create a VS to send all the traffic to v1 and mirror some traffic to v2
  $ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: helloworld-route
spec:
  hosts:
  - helloworld.default.svc.cluster.local
  http:
  - name: "helloworld-routes"
    route:
    - destination:
        host: helloworld.default.svc.cluster.local
        subset: v1
      weight: 100
    mirror:
      host: helloworld.default.svc.cluster.local
      subset: v2
    mirrorPercentage:
      value: 100
EOF
  # Let's test the  mirror capability. All the request should goes to the V2 of helloworld but the same number of request will be mirrored to V2.
  $ for i in {1..6}; do kubectl exec deploy/sleep -- curl -s helloworld:5000/hello; done
    Hello version: v1, instance: helloworld-v1-867747c89-6sqrw
    Hello version: v1, instance: helloworld-v1-867747c89-6sqrw
    Hello version: v1, instance: helloworld-v1-867747c89-6sqrw
    Hello version: v1, instance: helloworld-v1-867747c89-6sqrw
    Hello version: v1, instance: helloworld-v1-867747c89-6sqrw
    Hello version: v1, instance: helloworld-v1-867747c89-6sqrw
  # Check the logs for V2 helloworld
  $ kubectl logs -f -n default -l app=helloworld -l version=v2 -c istio-proxy
  [2024-01-28T08:49:59.442Z] "GET /hello HTTP/1.1" 200 - via_upstream - "-" 0 60 823 823 "10.42.0.6" "curl/8.5.0" "8d2adc94-7245-4054-b428-c6bfc8cad458" "helloworld-shadow:5000" "10.42.0.8:5000" inbound|5000|| 127.0.0.6:33893 10.42.0.8:5000 10.42.0.6:0 outbound_.5000_.v2_.helloworld.default.svc.cluster.local default
  [2024-01-28T08:50:00.371Z] "GET /hello HTTP/1.1" 200 - via_upstream - "-" 0 60 824 824 "10.42.0.6" "curl/8.5.0" "348c6426-d655-48a8-a0b7-75125ed467a7" "helloworld-shadow:5000" "10.42.0.8:5000" inbound|5000|| 127.0.0.6:54837 10.42.0.8:5000 10.42.0.6:0 outbound_.5000_.v2_.helloworld.default.svc.cluster.local default
  [2024-01-28T08:50:01.291Z] "GET /hello HTTP/1.1" 200 - via_upstream - "-" 0 60 827 827 "10.42.0.6" "curl/8.5.0" "5b624cea-77aa-4c18-8ec2-28898dc88a21" "helloworld-shadow:5000" "10.42.0.8:5000" inbound|5000|| 127.0.0.6:33893 10.42.0.8:5000 10.42.0.6:0 outbound_.5000_.v2_.helloworld.default.svc.cluster.local default
  [2024-01-28T08:50:02.206Z] "GET /hello HTTP/1.1" 200 - via_upstream - "-" 0 60 825 825 "10.42.0.6" "curl/8.5.0" "1de42867-b67c-40f0-ab60-176863b57440" "helloworld-shadow:5000" "10.42.0.8:5000" inbound|5000|| 127.0.0.6:33893 10.42.0.8:5000 10.42.0.6:0 outbound_.5000_.v2_.helloworld.default.svc.cluster.local default
  [2024-01-28T08:50:03.142Z] "GET /hello HTTP/1.1" 200 - via_upstream - "-" 0 60 825 825 "10.42.0.6" "curl/8.5.0" "20bfc8fb-3dc2-47bb-935d-22f1545f8070" "helloworld-shadow:5000" "10.42.0.8:5000" inbound|5000|| 127.0.0.6:54837 10.42.0.8:5000 10.42.0.6:0 outbound_.5000_.v2_.helloworld.default.svc.cluster.local default
  [2024-01-28T08:50:04.051Z] "GET /hello HTTP/1.1" 200 - via_upstream - "-" 0 60 822 821 "10.42.0.6" "curl/8.5.0" "7dc9901a-eddc-4bc6-bb4f-f3c46caa1223" "helloworld-shadow:5000" "10.42.0.8:5000" inbound|5000|| 127.0.0.6:54837 10.42.0.8:5000 10.42.0.6:0 outbound_.5000_.v2_.helloworld.default.svc.cluster.local default
  ```
</details>
