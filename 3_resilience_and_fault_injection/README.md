# Resilience and Fault Injection 20%

* [Configuring circuit breakers (with or without outlier detection)](#configuring-circuit-breakers-with-or-without-outlier-detection)
* [Using resilience features](#using-resilience-features)
* [Creating fault injection](#creating-fault-injection)

[Docs](https://istio.io/latest/docs/concepts/traffic-management/#network-resilience-and-testing)

## Configuring circuit breakers (with or without outlier detection)

[Task - Exam](https://istio.io/latest/docs/tasks/traffic-management/circuit-breaking/)

```bash
# Deploy sample httpbin and inject istio manually
$ kubectl apply -f <(istioctl kube-inject -f samples/httpbin/httpbin.yaml)

# Create a destination rule to apply circuit breaking settings when calling the httpbin service [with outlierDetection]

$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: httpbin
spec:
  host: httpbin
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
    outlierDetection:
      consecutive5xxErrors: 1
      interval: 1s
      baseEjectionTime: 3m
      maxEjectionPercent: 100
EOF

# Verify the destination rule was created correctly
$ kubectl get destinationrule httpbin -o yaml

# Create a client to send traffic to the httpbin service. 
# The client is a simple load-testing client called fortio.
# Fortio lets you control the number of connections, concurrency, and delays for outgoing HTTP calls.
# You will use this client to “trip” the circuit breaker policies you set in the DestinationRule.

$ kubectl apply -f <(istioctl kube-inject -f samples/httpbin/sample-client/fortio-deploy.yaml)

# Log in to the client pod and use the fortio tool to call httpbin. Pass in curl to indicate that you just want to make one call:
$ export FORTIO_POD=$(kubectl get pods -l app=fortio -o 'jsonpath={.items[0].metadata.name}')
$ kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio curl -quiet http://httpbin:8000/get
...
HTTP/1.1 200 OK
server: envoy
...
# You can see the request succeeded! Now, it’s time to break something.
# Tripping the circuit breaker
# In the DestinationRule settings, you specified maxConnections: 1 and http1MaxPendingRequests: 1.
# These rules indicate that if you exceed more than one connection and request concurrently, you should see some failures when the istio-proxy opens the circuit for further requests and connections.

# Call the service with two concurrent connections (-c 2) and send 20 requests (-n 20):
$ kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio load -c 2 -qps 0 -n 20 -loglevel Warning http://httpbin:8000/get
...
{"ts":1704298676.512727,"level":"info","r":1,"file":"logger.go","line":254,"msg":"Log level is now 3 Warning (was 2 Info)"}
Fortio 1.60.3 running at 0 queries per second, 4->4 procs, for 20 calls: http://httpbin:8000/get
Starting at max qps with 2 thread(s) [gomax 4] for exactly 20 calls (10 per thread + 0)
{"ts":1704298676.515909,"level":"warn","r":35,"file":"http_client.go","line":1104,"msg":"Non ok http code","code":503,"status":"HTTP/1.1 503","thread":0,"run":0}
...
# It’s interesting to see that almost all requests made it through! The istio-proxy does allow for some leeway.
Code 200 : 12 (60.0 %)
Code 503 : 8 (40.0 %)
...

# Bring the number of concurrent connections up to 3:
$ kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio load -c 3 -qps 0 -n 30 -loglevel Warning http://httpbin:8000/get
...
# Now you start to see the expected circuit breaking behavior. Only 33.3% of the requests succeeded and the rest were trapped by circuit breaking:
Code 200 : 10 (33.3 %)
Code 503 : 20 (66.7 %)
...

# Query the istio-proxy stats to see more
$ kubectl exec "$FORTIO_POD" -c istio-proxy -- pilot-agent request GET stats | grep httpbin | grep pending
cluster.outbound|8000||httpbin.default.svc.cluster.local.circuit_breakers.default.remaining_pending: 1
cluster.outbound|8000||httpbin.default.svc.cluster.local.circuit_breakers.default.rq_pending_open: 0
cluster.outbound|8000||httpbin.default.svc.cluster.local.circuit_breakers.high.rq_pending_open: 0
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_pending_active: 0
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_pending_failure_eject: 0
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_pending_overflow: 28
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_pending_total: 23

# You can see 21 for the upstream_rq_pending_overflow value which means 21 calls so far have been flagged for circuit breaking.

# Update the same destination rule to apply circuit breaking settings when calling the httpbin service [without outlierDetection]

#  In a circuit breaker, you set limits for calls to individual hosts within a service, such as the number of concurrent connections or how many times calls to this host have failed.
# Once that limit has been reached the circuit breaker “trips” and stops further connections to that host. 
# Using a circuit breaker pattern enables fast failure rather than clients trying to connect to an overloaded or failing host.

# The following example create a limit on the number of connections to 1 for TCP and HTTP
# When we reach that limit the circuit will be open "trip" and stops further connections to that host as a consecuence of 503 and upstream_rq_pending_overflow metric increase.
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: httpbin
spec:
  host: httpbin
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
EOF

# Bring the number of concurrent connections up to 3:
$ kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio load -c 3 -qps 0 -n 30 -loglevel Warning http://httpbin:8000/get

# Query the istio-proxy stats to see more
$ kubectl exec "$FORTIO_POD" -c istio-proxy -- pilot-agent request GET stats | grep httpbin | grep pending
cluster.outbound|8000||httpbin.default.svc.cluster.local.circuit_breakers.default.remaining_pending: 1
cluster.outbound|8000||httpbin.default.svc.cluster.local.circuit_breakers.default.rq_pending_open: 0
cluster.outbound|8000||httpbin.default.svc.cluster.local.circuit_breakers.high.rq_pending_open: 0
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_pending_active: 0
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_pending_failure_eject: 0
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_pending_overflow: 50
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_pending_total: 31

# You can see 50 for the upstream_rq_pending_overflow value which means 51 calls so far have been flagged for circuit breaking.
```

## Using resilience features

### Timeouts

[Docs](https://istio.io/latest/docs/tasks/traffic-management/request-timeouts/)

```bash
# A timeout for HTTP requests can be specified using a timeout field in a route rule. 
# By default, the request timeout is disabled, but in this task you override the reviews service timeout to half a second. 
# To see its effect, however, you also introduce an artificial 2 second delay in calls to the ratings service

# productpage -> reviews(0.5s) Request Timeout -> ratings(2s fault delay)

# Route requests to v2 of the reviews service, i.e., a version that calls the ratings service
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v2
EOF

# Add a 2 second delay to calls to the ratings service:
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - fault:
      delay:
        percentage:
          value: 100
        fixedDelay: 2s
    route:
    - destination:
        host: ratings
        subset: v1
EOF
# Open the Bookinfo web application in your browser
# You should see the Bookinfo application working normally (with ratings stars displayed), but there is a 2 second delay whenever you refresh the page.

# Now add a half second request timeout for calls to the reviews service:
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v2
    timeout: 0.5s
EOF
# Refresh the Bookinfo web page.
# You should now see that it returns in about 1 second, instead of 2, and the reviews are unavailable.
# The reason that the response takes 1 second, even though the timeout is configured at half a second, is because there is a hard-coded retry in the productpage service, so it calls the timing out reviews service twice before returning.

```

### Retries

[Docs](https://istio.io/latest/docs/concepts/traffic-management/#retries)

```bash
# A retry setting specifies the maximum number of times an Envoy proxy attempts to connect to a service if the initial call fails.

# In this example, all inbound requests to the helloworld v1 version service try 5 times, and an attempt is marked as failed if it takes longer than 1ms to complete.
# Deploy httpbin app
$ kubectl apply -f <(/Users/isaac.dominguez/Downloads/istio-1.18.2/bin/istioctl kube-inject -f samples/httpbin/httpbin.yaml)

$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: helloworld
spec:
  host: helloworld
  subsets:
  - name: v1
    labels:
      version: v1
EOF

$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: helloworld
spec:
  hosts:
  - helloworld
  http:
  - route:
    - destination:
        host: helloworld
        subset: v1
    retries:
      attempts: 5
      perTryTimeout: 1ms
      # https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/router_filter#x-envoy-retry-on
      retryOn: 5xx,reset,connect-failure,refused-stream
EOF

# Perform some request to helloworld to check the retries
$ for i in {1..100}; do
  kubectl exec deploy/sleep -- curl -vvvv -s helloworld:5000/hello
done

# In other terminal check the logs from the client who is performing the request deploy/sleep
# Enable log level to debug for the istio-proxy sidecar of deploy/sleep before
$ kubectl exec -it "$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})"  -c istio-proxy -- curl -X POST "localhost:15000/logging?level=debug"

# To see the 5 attempts of retries due the 1ms of request timeout trying to call helloworld
$ kubectl logs -f deploy/sleep -c istio-proxy
...
2024-01-04T15:04:29.919781Z	debug	envoy router external/envoy/source/common/router/router.cc:478	[C8298][S11051931266930981316] cluster 'outbound|5000|v1|helloworld.default.svc.cluster.local' match for URL '/hello'	thread=25
2024-01-04T15:04:29.919962Z	debug	envoy router external/envoy/source/common/router/router.cc:686	[C8298][S11051931266930981316] router decoding headers:
':authority', 'helloworld:5000'
':path', '/hello'
':method', 'GET'
':scheme', 'http'
'user-agent', 'curl/8.5.0'
'accept', '*/*'
'x-forwarded-proto', 'http'
'x-request-id', '05bad700-13e5-95e3-8538-401d8081cbf9'
'x-envoy-decorator-operation', 'helloworld.default.svc.cluster.local:5000/*'
'x-envoy-peer-metadata', 'XXX'
'x-envoy-peer-metadata-id', 'sidecar~10.42.0.11~sleep-58454f9847-xs64h.default~default.svc.cluster.local'
'x-envoy-expected-rq-timeout-ms', '1'
'x-envoy-attempt-count', '1'
	thread=25
...
2024-01-04T15:04:29.923515Z	debug	envoy router external/envoy/source/common/router/upstream_request.cc:602	[C8298][S11051931266930981316] upstream per try timeout	thread=25
...
2024-01-04T15:04:29.942749Z	debug	envoy router external/envoy/source/common/router/router.cc:1862	[C8298][S11051931266930981316] performing retry	thread=25
...
2024-01-04T15:04:29.946678Z	debug	envoy router external/envoy/source/common/router/upstream_request.cc:602	[C8298][S11051931266930981316] upstream per try timeout	thread=25
...
2024-01-04T15:04:29.970979Z	debug	envoy router external/envoy/source/common/router/router.cc:1862	[C8298][S11051931266930981316] performing retry	thread=25
...
2024-01-04T15:04:29.974518Z	debug	envoy router external/envoy/source/common/router/upstream_request.cc:602	[C8298][S11051931266930981316] upstream per try timeout	thread=25
...
2024-01-04T15:04:29.978682Z	debug	envoy router external/envoy/source/common/router/router.cc:1862	[C8298][S11051931266930981316] performing retry	thread=25
...
2024-01-04T15:04:29.982337Z	debug	envoy router external/envoy/source/common/router/upstream_request.cc:602	[C8298][S11051931266930981316] upstream per try timeout	thread=25
...
2024-01-04T15:04:30.097190Z	debug	envoy router external/envoy/source/common/router/router.cc:1862	[C8298][S11051931266930981316] performing retry	thread=25
...
2024-01-04T15:04:30.103283Z	debug	envoy router external/envoy/source/common/router/upstream_request.cc:602	[C8298][S11051931266930981316] upstream per try timeout	thread=25
...
2024-01-04T15:04:30.342979Z	debug	envoy router external/envoy/source/common/router/router.cc:1862	[C8298][S11051931266930981316] performing retry	thread=25
...
2024-01-04T15:04:30.346810Z	debug	envoy http external/envoy/source/common/http/filter_manager.cc:996	[C8298][S11051931266930981316] Sending local reply with details upstream_per_try_timeout	thread=25
2024-01-04T15:04:30.346837Z	debug	envoy http external/envoy/source/common/http/conn_manager_impl.cc:1680	[C8298][S11051931266930981316] encoding headers via codec (end_stream=false):
':status', '504'
'content-length', '24'
'content-type', 'text/plain'
'date', 'Thu, 04 Jan 2024 15:04:30 GMT'
'server', 'envoy'
	thread=25
2024-01-04T15:04:30.346845Z	debug	envoy http external/envoy/source/common/http/conn_manager_impl.cc:1772	[C8298][S11051931266930981316] Codec completed encoding stream.	thread=25
2024-01-04T15:04:30.347023Z	debug	envoy pool external/envoy/source/common/conn_pool/conn_pool_base.cc:215	[C8304] destroying stream: 0 remaining	thread=25
2024-01-04T15:04:30.348094Z	debug	envoy connection external/envoy/source/common/network/connection_impl.cc:656	[C8298] remote close	thread=25
2024-01-04T15:04:30.348110Z	debug	envoy connection external/envoy/source/common/network/connection_impl.cc:250	[C8298] closing socket: 0	thread=25
...

# You will see 5 times where it match with the     retries/attempts: 5:
# "performing retry"

# Check the how that VirtualService retries configuration is translate to Envoy configuration
$ kubectl exec -it "$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})"  -c istio-proxy -- curl -X GET "localhost:15000/config_dump" |grep -A14 "outbound|80|v1|helloworld.default.svc.cluster.local"

"cluster": "outbound|80|v1|helloworld.default.svc.cluster.local",
"timeout": "0s",
"retry_policy": {
"retry_on": "5xx,reset,connect-failure,refused-stream", # retryOn
"num_retries": 5, # attempts
"per_try_timeout": "0.001s", # perTryTimeout
"retry_host_predicate": [
  {
  "name": "envoy.retry_host_predicates.previous_hosts",
  "typed_config": {
    "@type": "type.googleapis.com/envoy.extensions.retry.host.previous_hosts.v3.PreviousHostsPredicate"
  }
  }
],
"host_selection_retry_max_attempts": "5"

```

## Creating fault injection

[Docs](https://istio.io/latest/docs/tasks/traffic-management/fault-injection/)

```bash
# Label the namespace that will host the application with istio-injection=enabled:
$ kubectl label namespace default istio-injection=enabled

# Deploy Bookinfo application using the kubectl command
$ kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml

# To confirm that the Bookinfo application is running, send a request to it by a curl command from some pod, for example from ratings:
$ kubectl exec "$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"

<title>Simple Bookstore App</title>

# Deploy the default destination rules, since we install istio using the demo profile we'll need to deploy the mtls.yaml since it's activated by default.
$ kubectl apply -f samples/bookinfo/networking/destination-rule-all-mtls.yaml

# Apply application version routing by running the following commands:
$ kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
$ kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml

# With the above configuration, this is how requests flow:

# productpage → reviews:v2 → ratings (only for user jason)
# productpage → reviews:v1 (for everyone else)

#
# Injecting an HTTP delay fault
#

# Create a fault injection rule to delay traffic coming from the test user jason
$ kubectl apply -f samples/bookinfo/networking/virtual-service-ratings-test-delay.yaml
# Confirm the rule was created:
$ kubectl get virtualservice ratings -o yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  generation: 2
  name: ratings
  namespace: default
spec:
  hosts:
  - ratings
  http:
  - fault:
      delay:
        fixedDelay: 7s
        percentage:
          value: 100
    match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: ratings
        subset: v1
  - route:
    - destination:
        host: ratings
        subset: v1
    
# Open the Bookinfo web application in your browser
# On the /productpage web page, log in as user jason.
# You expect the Bookinfo home page to load without errors in approximately 7 seconds. However, there is a problem: the Reviews section displays an error message:
*Sorry, product reviews are currently unavailable for this book.*
# Open the Developer Tools menu in you web browser.
# Open the Network tab
# Reload the /productpage web page. You will see that the page actually loads in about 6 seconds.

#
# Injecting an HTTP abort fault
#

# Create a fault injection rule to send an HTTP abort for user jason
$ kubectl get virtualservice ratings -o yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
...
spec:
  hosts:
  - ratings
  http:
  - fault:
      abort:
        httpStatus: 500
        percentage:
          value: 100
    match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: ratings
        subset: v1
  - route:
    - destination:
        host: ratings
        subset: v1
# Open the Bookinfo web application in your browser.
# On the /productpage, log in as user jason.
# If the rule propagated successfully to all pods, the page loads immediately and the Ratings service is currently unavailable message appears.
# If you log out from user jason or open the Bookinfo application in an anonymous window (or in another browser), you will see that /productpage still calls reviews:v1 (which does not call ratings at all) for everybody but jason. Therefore you will not see any error message.
```




