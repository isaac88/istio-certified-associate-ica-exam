# Securing Workloads 20%

* [Understand Istio security features](#understand-istio-security-features)
* [Set up Istio authorization for HTTP/TCP traffic in the mesh](#set-up-istio-authorization-for-httptcp-traffic-in-the-mesh)
* [Configure mutual TLS at mesh, namespace, and workload levels](#configure-mutual-tls-at-mesh-namespace-and-workload-levels)

[Docs](https://istio.io/latest/docs/concepts/security/)
[Configuration Reference](https://istio.io/latest/docs/reference/config/security/)

## Ex1: Understand Istio security features

[Docs](https://istio.io/latest/docs/concepts/security/)

Requirements:
* Read and understand:
  * Istio identity
  * Identity and certificate management
  * Authentication
    * [Mutual TLS authentication](https://istio.io/latest/docs/concepts/security/#mutual-tls-authentication)
    * [Authentication policies](https://istio.io/latest/docs/concepts/security/#authentication-policies)
  * Authorization
    * [Authorization policies](https://istio.io/latest/docs/concepts/security/#authorization-policies)

## Ex:2 Set up Istio authorization for HTTP/TCP traffic in the mesh

[Task](https://istio.io/latest/docs/tasks/security/authorization/)

### Ex:2.1 Authz HTTP

Requirements:
* Deploy [BookInfo app](https://raw.githubusercontent.com/istio/istio/release-1.20/samples/bookinfo/platform/kube/bookinfo.yaml)
* Create a allow-nothing policy in the default namespace
* Create a productpage-viewer policy to allow access with GET method to the productpage workload
* Create the details-viewer policy to allow the productpage workload with GET method
* Create the reviews-viewer policy to allow the productpage workload with GET methods
* Create the ratings-viewer policy to allow the reviews workload with GET methods
* Check the BookInfo app it should works and you should see the “black” and “red” ratings in the “Book Reviews” section.

<details>
  <summary>Solution</summary>

  [Task](https://istio.io/latest/docs/tasks/security/authorization/authz-http/)

  ```bash

  # Install Bookinfo App
  $ kubectl label namespace default istio-injection=enabled
  $ kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
  $ kubectl get services
  $ kubectl get pods
  $ kubectl exec "$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"
  $ kubectl port-forward service/productpage 9080:9080 -n default
  $ curl -s "http://localhost:9080/productpage" | grep -o "<title>.*</title>"
  $ kubectl apply -f samples/bookinfo/networking/destination-rule-all-mtls.yaml # "[istioctl install --set profile=demo]"

  # Configure access control for workloads using HTTP traffic
  # Run the following command to create a allow-nothing policy in the default namespace.
  $ kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: allow-nothing
  namespace: default
spec:
  {}
EOF

  # Run the following command to create a productpage-viewer policy to allow access with GET method to the productpage workload.
  # The policy does not set the from field in the rules which means all sources are allowed, effectively allowing all users and workloads.
  $ kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: "productpage-viewer"
  namespace: default
spec:
  selector:
    matchLabels:
      app: productpage
  action: ALLOW
  rules:
  - to:
    - operation:
        methods: ["GET"]
EOF

  # Point your browser at the Bookinfo productpage (http://localhost:9080/productpage). 
  # Now you should see the “Bookinfo Sample” page. However, you can see the following errors on the page:

  # Error fetching product details
  # Error fetching product reviews on the page.
  # These errors are expected because we have not granted the productpage workload access to the details and reviews workloads. 
  # Next, you need to configure a policy to grant access to those workloads.

  # Run the following command to create the details-viewer policy to allow the productpage workload, which issues requests using the cluster.local/ns/default/sa/bookinfo-productpage service account, to access the details workload through GET methods:
  $ kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: "details-viewer"
  namespace: default
spec:
  selector:
    matchLabels:
      app: details
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/default/sa/bookinfo-productpage"]
    to:
    - operation:
        methods: ["GET"]
EOF

  # Run the following command to create a policy reviews-viewer to allow the productpage workload, which issues requests using the cluster.local/ns/default/sa/bookinfo-productpage service account, to access the reviews workload through GET methods:
  $ kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: "reviews-viewer"
  namespace: default
spec:
  selector:
    matchLabels:
      app: reviews
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/default/sa/bookinfo-productpage"]
    to:
    - operation:
        methods: ["GET"]
EOF

  # Point your browser at the Bookinfo productpage (http://localhost:9080/productpage). 
  # Now, you should see the “Bookinfo Sample” page with “Book Details” on the lower left part, and “Book Reviews” on the lower right part. 
  # However, in the “Book Reviews” section, there is an error Ratings service currently unavailable.

  # This is because the reviews workload doesn’t have permission to access the ratings workload. 
  # To fix this issue, you need to grant the reviews workload access to the ratings workload. 
  # Next, we configure a policy to grant the reviews workload that access.

  # Run the following command to create the ratings-viewer policy to allow the reviews workload, which issues requests using the cluster.local/ns/default/sa/bookinfo-reviews service account, to access the ratings workload through GET methods.

  # Point your browser at the Bookinfo productpage (http://localhost:9080/productpage). 
  # You should see the “black” and “red” ratings in the “Book Reviews” section.

  # Congratulations! You successfully applied authorization policy to enforce access control for workloads using HTTP traffic.
  ```
</details>

### Ex2.2 Authz TCP

Requirements:
* Deploy tcp-echo service in the mesh in the namespace foo
* Deploy sleep service in the mesh in the namespace foo
* Create a tcp-policy authorization policy for the tcp-echo workload in the foo namespace to allow requests to port 9000 and 9001
* Verify that sleep service can communicate
* Update the existing tcp-policy to add an HTTP-only field GET named methods for port 9000
* Verify that sleep service can't communicate
* Update the existing tcp-policy to add a DENY policy with HTTP-only fields GET
* Verify that requests to port 9000 are denied
* Update the existing tcp-policy to add a DENY policy with HTTP-only fields GET for port 9000
* Verify that requests to port 9001 are allowed

<details>
  <summary>How to verify the connectivity</summary>

  ```bash
  # Verify that sleep successfully communicates with tcp-echo on ports 9000 and 9001 using the following command:
  $ kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" \
      -c sleep -n foo -- sh -c \
      'echo "port 9000" | nc tcp-echo 9000' | grep "hello" && echo 'connection succeeded' || echo 'connection rejected'
  hello port 9000
  connection succeeded

  $ kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" \
      -c sleep -n foo -- sh -c \
      'echo "port 9001" | nc tcp-echo 9001' | grep "hello" && echo 'connection succeeded' || echo 'connection rejected'
  hello port 9001
  connection succeeded

  # Verify that sleep successfully communicates with tcp-echo on port 9002. 
  # You need to send the traffic directly to the pod IP of tcp-echo because the port 9002 is not defined in the Kubernetes service object of tcp-echo. 
  # Get the pod IP address and send the request with the following command:
  $ TCP_ECHO_IP=$(kubectl get pod "$(kubectl get pod -l app=tcp-echo -n foo -o jsonpath={.items..metadata.name})" -n foo -o jsonpath="{.status.podIP}")
  $ kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" -c istio-proxy -n foo -- sh -c "echo 'port 9002' | nc $TCP_ECHO_IP 9002" | grep "hello"
  hello port 9002
  ```
</details>

<details>
  <summary>Solution</summary>

  [Task](https://istio.io/latest/docs/tasks/security/authorization/authz-tcp/)

  ```bash
  # Deploy two workloads named sleep and tcp-echo together in a namespace, for example foo. 
  # Both workloads run with an Envoy proxy in front of each. 
  # The tcp-echo workload listens on port 9000, 9001 and 9002 and echoes back any traffic it received with a prefix hello. 
  # For example, if you send “world” to tcp-echo, it will reply with hello world.
  #  The tcp-echo Kubernetes service object only declares the ports 9000 and 9001, and omits the port 9002. 
  # A pass-through filter chain will handle port 9002 traffic.

# Configure ALLOW authorization policy for a TCP workload
# Create the tcp-policy authorization policy for the tcp-echo workload in the foo namespace. 
# Run the following command to apply the policy to allow requests to port 9000 and 9001:
$ kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: tcp-policy
  namespace: foo
spec:
  selector:
    matchLabels:
      app: tcp-echo
  action: ALLOW
  rules:
  - to:
    - operation:
        ports: ["9000", "9001"]
EOF

# Verify that requests to port 9000 are allowed using the following command:
$ kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" \
    -c sleep -n foo -- sh -c \
    'echo "port 9000" | nc tcp-echo 9000' | grep "hello" && echo 'connection succeeded' || echo 'connection rejected'
hello port 9000
connection succeeded

# Verify that requests to port 9001 are allowed using the following command:
$ kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" \
    -c sleep -n foo -- sh -c \
    'echo "port 9001" | nc tcp-echo 9001' | grep "hello" && echo 'connection succeeded' || echo 'connection rejected'
hello port 9001
connection succeeded

# Verify that requests to port 9002 are denied. 
# This is enforced by the authorization policy which also applies to the pass through filter chain, even if the port is not declared explicitly in the tcp-echo Kubernetes service object. 
# Run the following command and verify the output:
$ kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" \
    -c sleep -n foo -- sh -c \
    "echo \"port 9002\" | nc $TCP_ECHO_IP 9002" | grep "hello" && echo 'connection succeeded' || echo 'connection rejected'
connection rejected

# Update the policy to add an HTTP-only field named methods for port 9000 using the following command:
$ kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: tcp-policy
  namespace: foo
spec:
  selector:
    matchLabels:
      app: tcp-echo
  action: ALLOW
  rules:
  - to:
    - operation:
        methods: ["GET"]
        ports: ["9000"]
EOF

# Verify that requests to port 9000 are denied.
# This occurs because the rule becomes invalid when it uses an HTTP-only field (methods) for TCP traffic. 
# Istio ignores the invalid ALLOW rule. 
# The final result is that the request is rejected, because it does not match any ALLOW rules. 
# Run the following command and verify the output:
$ kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" \
    -c sleep -n foo -- sh -c \
    'echo "port 9000" | nc tcp-echo 9000' | grep "hello" && echo 'connection succeeded' || echo 'connection rejected'
connection rejected

# Verify that requests to port 9001 are denied. This occurs because the requests do not match any ALLOW rules. 
# Run the following command and verify the output:
$ kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" \
    -c sleep -n foo -- sh -c \
    'echo "port 9001" | nc tcp-echo 9001' | grep "hello" && echo 'connection succeeded' || echo 'connection rejected'
connection rejected

# Configure DENY authorization policy for a TCP workload
# Add a DENY policy with HTTP-only fields using the following command:
$ kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: tcp-policy
  namespace: foo
spec:
  selector:
    matchLabels:
      app: tcp-echo
  action: DENY
  rules:
  - to:
    - operation:
        methods: ["GET"]
EOF
# Warning: configured AuthorizationPolicy will deny all traffic to TCP ports under its scope due to the use of only HTTP attributes in a DENY rule; it is recommended to explicitly specify the port
# authorizationpolicy.security.istio.io/tcp-policy configured

# Verify that requests to port 9000 are denied. 
# This occurs because Istio doesn’t understand the HTTP-only fields while creating a DENY rule for tcp port and due to it’s restrictive nature it denies all the traffic to the tcp ports:
$ kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" \
    -c sleep -n foo -- sh -c \
    'echo "port 9000" | nc tcp-echo 9000' | grep "hello" && echo 'connection succeeded' || echo 'connection rejected'
connection rejected

$ kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" \
    -c sleep -n foo -- sh -c \
    'echo "port 9001" | nc tcp-echo 9001' | grep "hello" && echo 'connection succeeded' || echo 'connection rejected'
connection rejected

# Add a DENY policy with both TCP and HTTP fields using the following command:
$ kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: tcp-policy
  namespace: foo
spec:
  selector:
    matchLabels:
      app: tcp-echo
  action: DENY
  rules:
  - to:
    - operation:
        methods: ["GET"]
        ports: ["9000"]
EOF

# Verify that requests to port 9000 is denied. 
# This occurs because the request matches the ports in the above-mentioned deny policy.
$ kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" \
    -c sleep -n foo -- sh -c \
    'echo "port 9000" | nc tcp-echo 9000' | grep "hello" && echo 'connection succeeded' || echo 'connection rejected'
connection rejected

# Verify that requests to port 9001 are allowed. 
# This occurs because the requests do not match the ports in the DENY policy:
$ kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" \
    -c sleep -n foo -- sh -c \
    'echo "port 9001" | nc tcp-echo 9001' | grep "hello" && echo 'connection succeeded' || echo 'connection rejected'
hello port 9001
connection succeeded
  ```
</details>


## Ex:3 Configure mutual TLS at mesh, namespace, and workload levels

[Task](https://istio.io/latest/docs/tasks/security/authentication/mtls-migration/)

*[Authentication Policy (check before)](https://istio.io/latest/docs/tasks/security/authentication/authn-policy/)*

*[PeerAuthentication](https://istio.io/latest/docs/reference/config/security/peer_authentication/)*

### Ex:3.1 Configure mutual TLS at namespace level

Requirements:
* Deploy httpbin service in the mesh in the foo namespace
* Deploy sleep service in the mesh in the foo namespace
* Deploy httpbin service in the mesh in the bar namespace
* Deploy sleep service in the mesh in the bar namespace
* Deploy sleep service NOT in the mesh in the legacy namespace
* Verify the setup by sending http requests (using curl) from the sleep pods, in namespaces foo, bar and legacy, to httpbin.foo and httpbin.bar. All requests should succeed with return code 200.
* Enforce mtls traffic on the foo namespace
* Verify that request comming from legacy namespace to the service on the foo namespace will fail
* Update the previous mtls traffic enforcement to be PERMISSIVE now and check the traffic with tcpdump you should see plain text traffic comming from legacy namespace


<details>
  <summary>How to verify the connectivity</summary>

  ```bash
  # Verify the setup by sending http requests (using curl) from the sleep pods, in namespaces foo, bar and legacy, to httpbin.foo and httpbin.bar. All requests should succeed with return code 200.
  $ for from in "foo" "bar" "legacy"; do for to in "foo" "bar"; do kubectl exec "$(kubectl get pod -l app=sleep -n ${from} -o jsonpath={.items..metadata.name})" -c sleep -n ${from} -- curl http://httpbin.${to}:8000/ip -s -o /dev/null -w "sleep.${from} to httpbin.${to}: %{http_code}\n"; done; done
  sleep.foo to httpbin.foo: 200
  sleep.foo to httpbin.bar: 200
  sleep.bar to httpbin.foo: 200
  sleep.bar to httpbin.bar: 200
  sleep.legacy to httpbin.foo: 200
  sleep.legacy to httpbin.bar: 200

  # Verify that now from the legacy namespace since no sidecars are inecjeted the requests should fail with return code 56 and namespace foo and bar should work.
  $ for from in "foo" "bar" "legacy"; do for to in "foo" "bar"; do kubectl exec "$(kubectl get pod -l app=sleep -n ${from} -o jsonpath={.items..metadata.name})" -c sleep -n ${from} -- curl http://httpbin.${to}:8000/ip -s -o /dev/null -w "sleep.${from} to httpbin.${to}: %{http_code}\n"; done; done
  sleep.foo to httpbin.foo: 200
  sleep.foo to httpbin.bar: 200
  sleep.bar to httpbin.foo: 200
  sleep.bar to httpbin.bar: 200
  sleep.legacy to httpbin.foo: 000 # Request failing since mTLS is enforced on foo namespace and no plain text traffic is allowed anymore.
  command terminated with exit code 56
  sleep.legacy to httpbin.bar: 200

  # Check if the traffic is encripted or not
  # Upgrade istio installation to have istio-proxy container as privileged container to use tcpump
  $ istioctl upgrade --set profile=demo --set values.global.proxy.privileged=true

  # Restart httpbin deployment
  $ kubectl rollout restart deploy httpbin -n foo

  # tcpdump on httpbin istio-proxy container
  $ kubectl exec -nfoo "$(kubectl get pod -nfoo -lapp=httpbin -ojsonpath={.items..metadata.name})" -c istio-proxy -- sudo tcpdump dst port 80  -A

  # Send again http requests (using curl) from the sleep pods, in namespaces foo, bar and legacy, to httpbin.foo and httpbin.bar. All requests should succeed with return code 200 less 
  # sleep.legacy to httpbin.foo: 000 since mtls mode: STRICT
  $ for from in "foo" "bar" "legacy"; do for to in "foo" "bar"; do kubectl exec "$(kubectl get pod -l app=sleep -n ${from} -o jsonpath={.items..metadata.name})" -c sleep -n ${from} -- curl http://httpbin.${to}:8000/ip -s -o /dev/null -w "sleep.${from} to httpbin.${to}: %{http_code}\n"; done; done
  # In the output of the tcpdump won't see any network data related to sleep.legacy to httpbin.foo
  ```
</details>

<details>
  <summary>Solution</summary>

  ```bash
  # Lock down to mutual TLS by namespace
  # After migrating all clients to Istio and injecting the Envoy sidecar, you can lock down workloads in the foo namespace to only accept mutual TLS traffic.
  $ kubectl apply -n foo -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: foo
spec:
  mtls:
    mode: STRICT
EOF

  # Now change to PERMISSIVE mode instead and perform that same request
  $ kubectl apply -n foo -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: foo
spec:
  mtls:
    mode: PERMISSIVE
EOF

  $ for from in "foo" "bar" "legacy"; do for to in "foo" "bar"; do kubectl exec "$(kubectl get pod -l app=sleep -n ${from} -o jsonpath={.items..metadata.name})" -c sleep -n ${from} -- curl http://httpbin.${to}:8000/ip -s -o /dev/null -w "sleep.${from} to httpbin.${to}: %{http_code}\n"; done; done

  # Now, you should see a similar oputput, it's a plain text data network traffic.
  # For the others that they are talking in mtls by default you can't read the same information since it's encripted.
  ...
  20:30:38.058445 IP 10-42-0-17.sleep.legacy.svc.cluster.local.57594 > httpbin-c558746df-xgx42.http: Flags [P.], seq 0:81, ack 1, win 507, options [nop,nop,TS val 652705005 ecr 3123133169], length 81: HTTP: GET /ip HTTP/1.1
  E...5F@.@...
  *..
  *.....P.=g................
  &.|..:.GET /ip HTTP/1.1
  Host: httpbin.foo:8000
  User-Agent: curl/8.5.0
  Accept: */*
  ...
  ```
</details>

### Ex:3.2 Configure mutual TLS at mesh level

Requirements:
* Enforce mtls traffic at mesh global level
* Be sure that you remove the previous PeerAuthentication from the foo namespace
* Verify that request coming from legacy namespace doesn't work

<details>
  <summary>How to verify the connectivity</summary>

  ```bash
  # Now, both the foo and bar namespaces enforce mutual TLS only traffic, so you should see requests from sleep.legacy failing for both.
  $ for from in "foo" "bar" "legacy"; do for to in "foo" "bar"; do kubectl exec "$(kubectl get pod -l app=sleep -n ${from} -o jsonpath={.items..metadata.name})" -c sleep -n ${from} -- curl http://httpbin.${to}:8000/ip -s -o /dev/null -w "sleep.${from} to httpbin.${to}: %{http_code}\n"; done; done
  sleep.foo to httpbin.foo: 200
  sleep.foo to httpbin.bar: 200
  sleep.bar to httpbin.foo: 200
  sleep.bar to httpbin.bar: 200
  sleep.legacy to httpbin.foo: 000
  command terminated with exit code 56
  sleep.legacy to httpbin.bar: 000
  command terminated with exit code 56
  ```
</details>

<details>
  <summary>Solution</summary>

  ```bash
  # Lock down mutual TLS for the entire mesh
  # You can lock down workloads in all namespaces to only accept mutual TLS traffic by putting the policy in the system namespace of your Istio installation.
  $ kubectl apply -n istio-system -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
EOF

  # Now, both the foo and bar namespaces enforce mutual TLS only traffic, so you should see requests from sleep.legacy failing for both.
  $ for from in "foo" "bar" "legacy"; do for to in "foo" "bar"; do kubectl exec "$(kubectl get pod -l app=sleep -n ${from} -o jsonpath={.items..metadata.name})" -c sleep -n ${from} -- curl http://httpbin.${to}:8000/ip -s -o /dev/null -w "sleep.${from} to httpbin.${to}: %{http_code}\n"; done; done
  sleep.foo to httpbin.foo: 200
  sleep.foo to httpbin.bar: 200
  sleep.bar to httpbin.foo: 200
  sleep.bar to httpbin.bar: 200
  sleep.legacy to httpbin.foo: 000
  command terminated with exit code 56
  sleep.legacy to httpbin.bar: 000
  command terminated with exit code 56
  ```
</details>

### Ex:3.3 Configure mutual TLS at workload level

Rquirements:
* We have at mesh level set the strict mode
* Create a policy to allow both mTLS & plaintext traffic for httpbin on foo namespace
* Create a policy to allow both mTLS & plaintext traffic for httpbin on bar namespace
* Verify that request coming from sleep pods in all the namespace to httpbin in foo and bar namespace will work

<details>
  <summary>How to verify the connectivity</summary>

  ```bash
  # Perform again the same request, all the request should pass.

  $ for from in "foo" "bar" "legacy"; do for to in "foo" "bar"; do kubectl exec "$(kubectl get pod -l app=sleep -n ${from} -o jsonpath={.items..metadata.name})" -c sleep -n ${from} -- curl http://httpbin.${to}:8000/ip -s -o /dev/null -w "sleep.${from} to httpbin.${to}: %{http_code}\n"; done; done
  sleep.foo to httpbin.foo: 200
  sleep.foo to httpbin.bar: 200
  sleep.bar to httpbin.foo: 200
  sleep.bar to httpbin.bar: 200
  sleep.legacy to httpbin.foo: 200
  sleep.legacy to httpbin.bar: 200
  ```
</details>

<details>
  <summary>Solution</summary>

  ```bash
  # We have at mesh level set the strict mode but we'll override it for specific worloads on specifics namespaces.
  # Create a policy to allow both mTLS & plaintext traffic for httpbin on foo namespace
  $ kubectl apply -n foo -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: httpbin
  namespace: foo
spec:
  selector:
    matchLabels:
      app: httpbin
  mtls:
    mode: PERMISSIVE
EOF

  # Create a policy to allow both mTLS & plaintext traffic for httpbin on bar namespace
  $ kubectl apply -n bar -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: httpbin
  namespace: bar
spec:
  selector:
    matchLabels:
      app: httpbin
  mtls:
    mode: PERMISSIVE
EOF
  ```
</details>
