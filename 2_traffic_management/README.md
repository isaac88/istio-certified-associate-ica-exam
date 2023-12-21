# Traffic Management 40%

* Controlling network traffic flows within a service mesh
* Configuring sidecar injection
* Using the Gateway resource to configure ingress and egress traffic
* Understanding how to use ServiceEntry resources for adding entries to internal service registry
* Define traffic policies using DestinationRule
* Configure traffic mirroring capabilities

[Docs](https://istio.io/latest/docs/concepts/traffic-management/)

[Task - Exam](https://istio.io/latest/docs/tasks/traffic-management/)

## Configuring sidecar injection

[Docs](https://istio.io/latest/docs/setup/additional-setup/sidecar-injection/)

[Docs](https://tetratelabs.github.io/istio-0to60/sidecar-injection/)

```bash

# Label the default namespace with istio-injection=enabled

$ kubectl label namespace default istio-injection=enabled --overwrite

# Disable injection for the default namespace
$ kubectl label namespace default istio-injection-

# Manual sidecar injection using cluster configurations

$ istioctl kube-inject -f samples/sleep/sleep.yaml | kubectl apply -f -

# Create a persistent version of the deployment with Istio sidecar injected.
$ istioctl kube-inject -f samples/sleep/sleep.yaml -o sleep-injected.yaml

# Update an existing deployment.
$ kubectl get deployment -o yaml | istioctl kube-inject -f - | kubectl apply -f -

# Manual sidecar injection using local configurations
# This can be get it from istioctl kube-inject --help
$ kubectl -n istio-system get configmap istio-sidecar-injector -o=jsonpath='{.data.config}' > inject-config.yaml
$ kubectl -n istio-system get configmap istio-sidecar-injector -o=jsonpath='{.data.values}' > inject-values.yaml
$ kubectl -n istio-system get configmap istio -o=jsonpath='{.data.mesh}' > mesh-config.yaml

$ istioctl kube-inject \
    --injectConfigFile inject-config.yaml \
    --meshConfigFile mesh-config.yaml \
    --valuesFile inject-values.yaml \
    --filename samples/sleep/sleep.yaml \
    | kubectl apply -f -
```

## Using the Gateway resource to configure ingress and egress traffic

[Docs Gateway](https://istio.io/latest/docs/reference/config/networking/gateway/)

### Gateway: Ingress

[Task - Ingress](https://istio.io/latest/docs/tasks/traffic-management/ingress/)

```bash

# Deploy demo app httbin
$ kubectl apply -f samples/httpbin/httpbin.yaml

# Create Istio Ingress Gateway
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: httpbin-gateway
spec:
  # The selector matches the ingress gateway pod labels.
  # If you installed Istio using Helm following the standard documentation, this would be "istio=ingress"
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "httpbin.example.com"
EOF

# Configure routes for traffic entering via the Gateway

kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
  - "httpbin.example.com"
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
          number: 8000
        host: httpbin
EOF

# Access the httpbin service using curl
$ kubectl port-forward service/istio-ingressgateway 8080:80 -n istio-system

$ curl -s -I -HHost:httpbin.example.com "http://localhost:8080/status/200"
HTTP/1.1 200 OK

```

### Gateway: Egress

[Task - Egress](https://istio.io/latest/docs/tasks/traffic-management/egress/egress-gateway/)

```bash

# Deploy the sleep sample app to use as a test source for sending requests.
# Manual sidecar injection since automatic injection is not enabled.
$ kubectl apply -f <(istioctl kube-inject -f samples/sleep/sleep.yaml)

# Set the SOURCE_POD environment variable to the name of your source pod:
$ export SOURCE_POD=$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})

# Enable Envoy's access logging
# https://istio.io/latest/docs/tasks/observability/logs/access-log/#enable-envoys-access-logging

# Using Telemetry API
kubectl apply -f - <<EOF
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: mesh-default
  namespace: istio-system
spec:
  accessLogging:
    - providers:
      - name: envoy
EOF

# Check if the Istio egress gateway is deployed:
$ kubectl get pod -l istio=egressgateway -n istio-system

# Egress gateway for HTTP traffic
# First create a ServiceEntry to allow direct traffic to an external service
# Define a ServiceEntry for edition.cnn.com.
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: cnn
spec:
  hosts:
  - edition.cnn.com
  ports:
  - number: 80
    name: http-port
    protocol: HTTP
  - number: 443
    name: https
    protocol: HTTPS
  resolution: DNS
EOF

# Verify that your ServiceEntry was applied correctly by sending an HTTP request to http://edition.cnn.com/politics
$ kubectl exec "$SOURCE_POD" -c sleep -- curl -sSL -o /dev/null -D - http://edition.cnn.com/politics
...
HTTP/1.1 301 Moved Permanently
...
location: https://edition.cnn.com/politics
...

HTTP/2 200
Content-Type: text/html; charset=utf-8
...

# Create an egress Gateway for edition.cnn.com, port 80, and a destination rule for traffic directed to the egress gateway.
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: istio-egressgateway
spec:
  selector:
    istio: egressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - edition.cnn.com
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: egressgateway-for-cnn
spec:
  host: istio-egressgateway.istio-system.svc.cluster.local
  subsets:
  - name: cnn
EOF

# Define a VirtualService to direct traffic from the sidecars to the egress gateway and from the egress gateway to the external service:
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: direct-cnn-through-egress-gateway
spec:
  hosts:
  - edition.cnn.com
  gateways:
  - istio-egressgateway
  - mesh
  http:
  - match:
    - gateways:
      - mesh
      port: 80
    route:
    - destination:
        host: istio-egressgateway.istio-system.svc.cluster.local
        subset: cnn
        port:
          number: 80
      weight: 100
  - match:
    - gateways:
      - istio-egressgateway
      port: 80
    route:
    - destination:
        host: edition.cnn.com
        port:
          number: 80
      weight: 100
EOF

# Check the log of the istio-egressgateway pod for a line corresponding to our request. If Istio is deployed in the istio-system namespace, the command to print the log is
$ kubectl logs -l istio=egressgateway -c istio-proxy -n istio-system | tail


# Egress gateway for HTTPS traffic
# https://istio.io/latest/docs/tasks/traffic-management/egress/egress-gateway/#egress-gateway-for-https-traffic
# In this section you direct HTTPS traffic (TLS originated by the application) through an egress gateway

# Define a ServiceEntry for edition.cnn.com
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: cnn
spec:
  hosts:
  - edition.cnn.com
  ports:
  - number: 443
    name: tls
    protocol: TLS
  resolution: DNS
EOF

# Verify that your ServiceEntry was applied correctly by sending an HTTPS request to https://edition.cnn.com/politics
$ kubectl exec "$SOURCE_POD" -c sleep -- curl -sSL -o /dev/null -D - https://edition.cnn.com/politics
...
HTTP/2 200
Content-Type: text/html; charset=utf-8
...

# Create an egress Gateway for edition.cnn.com, a destination rule and a virtual service to direct the traffic through the egress gateway and from the egress gateway to the external service.
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: istio-egressgateway
spec:
  selector:
    istio: egressgateway
  servers:
  - port:
      number: 443
      name: tls
      protocol: TLS
    hosts:
    - edition.cnn.com
    tls:
      mode: PASSTHROUGH
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: egressgateway-for-cnn
spec:
  host: istio-egressgateway.istio-system.svc.cluster.local
  subsets:
  - name: cnn
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: direct-cnn-through-egress-gateway
spec:
  hosts:
  - edition.cnn.com
  gateways:
  - mesh
  - istio-egressgateway
  tls:
  - match:
    - gateways:
      - mesh
      port: 443
      sniHosts:
      - edition.cnn.com
    route:
    - destination:
        host: istio-egressgateway.istio-system.svc.cluster.local
        subset: cnn
        port:
          number: 443
  - match:
    - gateways:
      - istio-egressgateway
      port: 443
      sniHosts:
      - edition.cnn.com
    route:
    - destination:
        host: edition.cnn.com
        port:
          number: 443
      weight: 100
EOF

# Send an HTTPS request to https://edition.cnn.com/politics. The output should be the same as before
$ kubectl exec "$SOURCE_POD" -c sleep -- curl -sSL -o /dev/null -D - https://edition.cnn.com/politics
...
HTTP/2 200
Content-Type: text/html; charset=utf-8
...

# Check the log of the egress gateway’s proxy. If Istio is deployed in the istio-system namespace, the command to print the log is
$ kubectl logs -l istio=egressgateway -n istio-system
[2024-01-01T15:47:08.221Z] "- - -" 0 - - - "-" 791 2502541 267 - "-" "-" "-" "-" "151.101.195.5:443" outbound|443||edition.cnn.com 10.42.0.25:43706 10.42.0.25:8443 10.42.0.21:53416 edition.cnn.com -

```

## Understanding how to use ServiceEntry resources for adding entries to internal service registry

[Docs](https://istio.io/latest/docs/reference/config/networking/service-entry/)

```bash

# Istio maintains an internal service registry which can be observed through a debug endpoint /debug/registryz exposed by istiod
$ kubectl exec -n istio-system deploy/istiod -- curl -s localhost:15014/debug/registryz

# The following example declares a few external APIs accessed by internal applications over HTTPS. The sidecar inspects the SNI value in the ClientHello message to route to the appropriate external service.

$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: external-svc-https
spec:
  hosts:
  - api.dropboxapi.com
  - www.googleapis.com
  - api.facebook.com
  location: MESH_EXTERNAL
  ports:
  - number: 443
    name: https
    protocol: TLS
  resolution: DNS
EOF

# Check if the envoy cluster "api.dropboxapi.com" endpoint is available for sleep pods. 
$ istioctl proxy-config endpoints deploy/sleep \
  --cluster "outbound|443||api.dropboxapi.com"
ENDPOINT              STATUS      OUTLIER CHECK     CLUSTER
162.125.68.19:443     HEALTHY     OK                outbound|443||api.dropboxapi.com 

# Check the external APIs are part of the registry as external services "MeshExternal": true
$ kubectl exec -n istio-system deploy/istiod -- \
  curl -s localhost:15014/debug/registryz | jq '.[] | select( .hostname | contains("mymongodb.somedomain"))'

# MESH_INTERNAL
# The following configuration adds a set of MongoDB instances running on unmanaged VMs to Istio’s registry, so that these services can be treated as any other service in the mesh. The associated DestinationRule is used to initiate mTLS connections to the database instances.

$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: external-svc-mongocluster
spec:
  hosts:
  - mymongodb.somedomain # not used
  addresses:
  - 192.192.192.192/24 # VIPs
  ports:
  - number: 27018
    name: mongodb
    protocol: MONGO
  location: MESH_INTERNAL
  resolution: STATIC
  endpoints:
  - address: 2.2.2.2
  - address: 3.3.3.3
EOF

# Check the new MongoDB instances are part of the registry as internal services "MeshExternal": false
$ kubectl exec -n istio-system deploy/istiod -- \
  curl -s localhost:15014/debug/registryz | jq '.[] | select( .hostname | contains("mymongodb.somedomain"))'

```

## Define traffic policies using DestinationRule

[Docs](https://istio.io/latest/docs/reference/config/networking/destination-rule/)

### Load balancing

[Load balancing](https://tetratelabs.github.io/istio-0to60/discovery/#load-balancing)

```bash

# Deploy helloworld application
$ kubectl apply -f samples/helloworld/helloworld.yaml

# Make repeated calls to the helloworld service from the sleep pod
$ for i in {1..6}; do
  kubectl exec deploy/sleep -- curl -s helloworld:5000/hello
done

# Examine the helloworld "cluster" definition in a sample client to see what load balancing policy is in effect:
$ istioctl proxy-config cluster deploy/sleep \
  --fqdn helloworld.default.svc.cluster.local -o yaml | grep lbPolicy
  lbPolicy: LEAST_REQUEST

# To influence the load balancing algorithm that Envoy uses when calling helloworld, we can define a traffic policy, like so:
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: helloworld-trafficpolicy
spec:
  host: helloworld.default.svc.cluster.local
  trafficPolicy:
    loadBalancer:
      simple: RANDOM
EOF

# Examine again the load balancing algorithm that Envoy uses when calling helloworld
$ istioctl proxy-config cluster deploy/sleep \
  --fqdn helloworld.default.svc.cluster.local -o yaml | grep lbPolicy
  lbPolicy: RANDOM
```

### Traffic distribution

[Traffic distribution](https://tetratelabs.github.io/istio-0to60/discovery/#traffic-distribution)

```bash

# We can go a step further and control how much traffic to send to version v1 and how much to v2
# First, define the two subsets, v1 and v2
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: helloworld-trafficpolicy
spec:
  host: helloworld.default.svc.cluster.local
  trafficPolicy:
    loadBalancer:
      simple: LEAST_REQUEST
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
EOF

# Inspect the list of clusters, note that there's one for each subset
$ istioctl proxy-config cluster deploy/sleep \
  --fqdn helloworld.default.svc.cluster.local
SERVICE FQDN                             PORT     SUBSET     DIRECTION     TYPE     DESTINATION RULE
helloworld.default.svc.cluster.local     5000     -          outbound      EDS      helloworld-trafficpolicy.default
helloworld.default.svc.cluster.local     5000     v1         outbound      EDS      helloworld-trafficpolicy.default
helloworld.default.svc.cluster.local     5000     v2         outbound      EDS      helloworld-trafficpolicy.default

# VirtualService, in this case to direct 25% of traffic to v1 and 75% to v2
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: hello-routing
spec:
  hosts:
  - helloworld.default.svc.cluster.local
  http:
  - route:
    - destination:
        host: helloworld.default.svc.cluster.local
        subset: v1
      weight: 25
    - destination:
        host: helloworld.default.svc.cluster.local
        subset: v2
      weight: 75
EOF

# Inspect the routing rules applied trough the VirtialService
$ istioctl proxy-config routes deploy/sleep --name 5000 -o yaml
...
        weightedClusters:
          clusters:
          - name: outbound|5000|v1|helloworld.default.svc.cluster.local
            weight: 25
          - name: outbound|5000|v2|helloworld.default.svc.cluster.local
            weight: 75
...

# istioctl CLI provides a convenient command to inspect the configuration of a service:
$ istioctl x describe svc helloworld
Service: helloworld
   Port: http 5000/HTTP targets pod port 5000
DestinationRule: helloworld-trafficpolicy for "helloworld.default.svc.cluster.local"
   Matching subsets: v1,v2
   load balancer
VirtualService: hello-routing
   Route to host "helloworld.default.svc.cluster.local" subset "v1" with weight 25%
   Route to host "helloworld.default.svc.cluster.local" subset "v2" with weight 75%

```


## Configure traffic mirroring capabilities

[Docs](https://istio.io/latest/docs/tasks/traffic-management/mirroring/)

```bash
# In this task, you will first force all traffic to v1 of a test service. Then, you will apply a rule to mirror a portion of traffic to v2.
# Start by deploying two versions of the httpbin service that have access logging enabled.
# httpbin-v1
$ cat <<EOF | istioctl kube-inject -f - | kubectl create -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpbin-v1
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
    spec:
      containers:
      - image: docker.io/kennethreitz/httpbin
        imagePullPolicy: IfNotPresent
        name: httpbin
        command: ["gunicorn", "--access-logfile", "-", "-b", "0.0.0.0:80", "httpbin:app"]
        ports:
        - containerPort: 80
EOF
# httpbin-v2
$ cat <<EOF | istioctl kube-inject -f - | kubectl create -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpbin-v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: httpbin
      version: v2
  template:
    metadata:
      labels:
        app: httpbin
        version: v2
    spec:
      containers:
      - image: docker.io/kennethreitz/httpbin
        imagePullPolicy: IfNotPresent
        name: httpbin
        command: ["gunicorn", "--access-logfile", "-", "-b", "0.0.0.0:80", "httpbin:app"]
        ports:
        - containerPort: 80
EOF
# httpbin Kubernetes service
$ kubectl create -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: httpbin
  labels:
    app: httpbin
spec:
  ports:
  - name: http
    port: 8000
    targetPort: 80
  selector:
    app: httpbin
EOF
# Start the sleep service so you can use curl to provide load
$ cat <<EOF | istioctl kube-inject -f - | kubectl create -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sleep
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sleep
  template:
    metadata:
      labels:
        app: sleep
    spec:
      containers:
      - name: sleep
        image: curlimages/curl
        command: ["/bin/sleep","3650d"]
        imagePullPolicy: IfNotPresent
EOF
# Create a default route rule to route all traffic to v1 of the service
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
    - httpbin
  http:
  - route:
    - destination:
        host: httpbin
        subset: v1
      weight: 100
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: httpbin
spec:
  host: httpbin
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
EOF
# Now, with all traffic directed to httpbin:v1, send a request to the service
$ export SLEEP_POD=$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})
$ kubectl exec "${SLEEP_POD}" -c sleep -- curl -sS http://httpbin:8000/headers
{
  "headers": {
    "Accept": "*/*",
    "Content-Length": "0",
    "Host": "httpbin:8000",
    "User-Agent": "curl/7.35.0",
    "X-B3-Parentspanid": "57784f8bff90ae0b",
    "X-B3-Sampled": "1",
    "X-B3-Spanid": "3289ae7257c3f159",
    "X-B3-Traceid": "b56eebd279a76f0b57784f8bff90ae0b",
    "X-Envoy-Attempt-Count": "1",
    "X-Forwarded-Client-Cert": "By=spiffe://cluster.local/ns/default/sa/default;Hash=20afebed6da091c850264cc751b8c9306abac02993f80bdb76282237422bd098;Subject=\"\";URI=spiffe://cluster.local/ns/default/sa/default"
  }
}
# Check the logs for v1 and v2 of the httpbin pods. You should see access log entries for v1 and none for v2
$ export V1_POD=$(kubectl get pod -l app=httpbin,version=v1 -o jsonpath={.items..metadata.name})
$ kubectl logs "$V1_POD" -c httpbin
...
127.0.0.6 - - [02/Jan/2024:10:07:21 +0000] "GET /headers HTTP/1.1" 200 522 "-" "curl/8.5.0"
...

$ export V2_POD=$(kubectl get pod -l app=httpbin,version=v2 -o jsonpath={.items..metadata.name})
$ kubectl logs "$V2_POD" -c httpbin
<none>

# Mirroring traffic to v2
# Change the route rule to mirror traffic to v2:
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
    - httpbin
  http:
  - route:
    - destination:
        host: httpbin
        subset: v1
      weight: 100
    mirror:
      host: httpbin
      subset: v2
    mirrorPercentage:
      value: 100.0
EOF
# We send traffic again:
$ for i in {1..20}; do
  kubectl exec deploy/sleep -- curl -s httpbin:8000/headers
done
# Check the logs now of the mirror destination httpbin-2
$ kubectl logs "$V2_POD" -c httpbin
...
127.0.0.6 - - [02/Jan/2024:10:20:00 +0000] "GET /headers HTTP/1.1" 200 562 "-" "curl/8.5.0"
...

```