# Advanced Scenarios 13%

* [Understand how to onboard non-Kubernetes workloads to the mesh](https://istio.io/latest/docs/ops/deployment/vm-architecture/)
* [Troubleshoot configuration issues](https://istio.io/latest/docs/ops/common-problems/)

## Understand how to onboard non-Kubernetes workloads to the mesh

[Docs](https://istio.io/latest/docs/ops/deployment/vm-architecture/)

### Onboard non-Kubernetes workloads to the mesh

[Task](https://istio.io/latest/docs/setup/install/virtual-machine/)

```bash
# This example application running on a virtual machine (VM), and illustrates how to control this infrastructure as a single mesh.

# You have already a candidate Virtual Machine created to onboard on the Istio Mesh

# Prepare the guide environment
# Set the environment variables VM_APP, WORK_DIR , VM_NAMESPACE, and SERVICE_ACCOUNT on your machine that you’re using to set up the cluster. (e.g., WORK_DIR="${HOME}/vmintegration"):
$ export VM_APP="vm-mysqldb"
$ export VM_NAMESPACE="vm-mysqldb"
$ export WORK_DIR="/tmp"
$ export SERVICE_ACCOUNT="vm-mysqldb"
$ export CLUSTER_NETWORK=""
$ export VM_NETWORK=""
$ export CLUSTER="Kubernetes"

# Create the working directory on your machine that you’re using to set up the cluster:
$ mkdir -p "${WORK_DIR}"

# We assume that you have already installed Istio.

# Deploy the east-west gateway. We'll use an empty profile to create it as new component.
$ samples/multicluster/gen-eastwest-gateway.sh --single-cluster | ./bin/istioctl install -y -f -
✔ Ingress gateways installed
✔ Installation complete
Thank you for installing Istio 1.16.  Please take a few minutes to tell us about your install/upgrade experience!  https://forms.gle/99uiMML96AmsXY5d6

# Expose services inside the cluster via the east-west gateway: Gateway & VirtualService
$ kubectl apply -n istio-system -f samples/multicluster/expose-istiod.yaml

# Configure the VM namespace
# Create the namespace that will host the virtual machine:
$ kubectl create namespace "${VM_NAMESPACE}"

# Create a serviceaccount for the virtual machine:
$ kubectl create serviceaccount "${SERVICE_ACCOUNT}" -n "${VM_NAMESPACE}"

# Create files to transfer to the virtual machine
# First, create a template WorkloadGroup for the VM(s):
$ cat <<EOF > workloadgroup.yaml
apiVersion: networking.istio.io/v1beta1
kind: WorkloadGroup
metadata:
  name: "${VM_APP}"
  namespace: "${VM_NAMESPACE}"
spec:
  metadata:
    labels:
      app: "${VM_APP}"
  template:
    serviceAccount: "${SERVICE_ACCOUNT}"
    network: "${VM_NETWORK}"
EOF

# Before proceeding to generate the istio-token, as part of istioctl x workload entry, you should verify third party tokens are enabled in your cluster by following the steps describe here. If third party tokens are not enabled, you should add the option --set values.global.jwtPolicy=first-party-jwt to the Istio install commands.
# If this returns no response, then the feature is not supported:
$ kubectl get --raw /api/v1 | jq '.resources[] | select(.name | index("serviceaccounts/token"))'
{
  "name": "serviceaccounts/token",
  "singularName": "",
  "namespaced": true,
  "group": "authentication.k8s.io",
  "version": "v1",
  "kind": "TokenRequest",
  "verbs": [
    "create"
  ]
}

# Use the istioctl x workload entry command to generate:

# cluster.env: Contains metadata that identifies what namespace, service account, network CIDR and (optionally) what inbound ports to capture.
# istio-token: A Kubernetes token used to get certs from the CA.
# mesh.yaml: Provides ProxyConfig to configure discoveryAddress, health-checking probes, and some authentication options.
# root-cert.pem: The root certificate used to authenticate.
# hosts: An addendum to /etc/hosts that the proxy will use to reach istiod for xDS.*
$ istioctl x workload entry configure -f workloadgroup.yaml -o "${WORK_DIR}" --clusterID "${CLUSTER}"
Warning: a security token for namespace "vm-mysqldb" and service account "vm-mysqldb" has been generated and stored at "/tmp/istio-token"
2024-01-11T15:42:54.729859Z	warn	Could not auto-detect IP for istiod.istio-system.svc/istio-system. Use --ingressIP to manually specify the Gateway address to reach istiod from the VM.
Configuration generation into directory /tmp was successful

# Configure the virtual machine
# Run the following commands on the virtual machine you want to add to the Istio mesh:

# Securely transfer the files from "${WORK_DIR}" to the virtual machine. How you choose to securely transfer those files should be done with consideration for your information security policies. For convenience in this guide, transfer all of the required files to "${HOME}" in the virtual machine.

# Install the root certificate at /etc/certs:
$ sudo mkdir -p /etc/certs
$ sudo cp "${HOME}"/root-cert.pem /etc/certs/root-cert.pem

# Install the token at /var/run/secrets/tokens:
$ sudo  mkdir -p /var/run/secrets/tokens
$ sudo cp "${HOME}"/istio-token /var/run/secrets/tokens/istio-token

# Install the package containing the Istio virtual machine integration runtime:
$ curl -LO https://storage.googleapis.com/istio-release/releases/1.18.2/deb/istio-sidecar.deb
$ sudo dpkg -i istio-sidecar.deb

# Install cluster.env within the directory /var/lib/istio/envoy/:
$ sudo cp "${HOME}"/cluster.env /var/lib/istio/envoy/cluster.env

# Install the Mesh Config to /etc/istio/config/mesh:
$ sudo cp "${HOME}"/mesh.yaml /etc/istio/config/mesh

# Add the istiod host to /etc/hosts:
$ sudo sh -c 'cat $(eval echo ~$SUDO_USER)/hosts >> /etc/hosts'

# Transfer ownership of the files in /etc/certs/ and /var/lib/istio/envoy/ to the Istio proxy:
$ sudo chown -R istio-proxy /var/lib/istio /etc/certs /etc/istio/proxy /etc/istio/config /var/run/secrets /etc/certs/root-cert.pem

# Start Istio within the virtual machine
# Start the Istio agent:
$ sudo systemctl start istio

# Verify Istio Works Successfully
$ tail /var/log/istio/istio.err.log /var/log/istio/istio.log -Fq -n 100

2024-01-11T18:42:00.412798Z	info	FLAG: --concurrency="0"
2024-01-11T18:42:00.412834Z	info	FLAG: --domain=""
2024-01-11T18:42:00.412842Z	info	FLAG: --help="false"
2024-01-11T18:42:00.412847Z	info	FLAG: --log_as_json="false"
2024-01-11T18:42:00.412851Z	info	FLAG: --log_caller=""
2024-01-11T18:42:00.412857Z	info	FLAG: --log_output_level="default:info"
2024-01-11T18:42:00.412861Z	info	FLAG: --log_rotate=""
2024-01-11T18:42:00.412866Z	info	FLAG: --log_rotate_max_age="30"
2024-01-11T18:42:00.412871Z	info	FLAG: --log_rotate_max_backups="1000"
2024-01-11T18:42:00.412876Z	info	FLAG: --log_rotate_max_size="104857600"
2024-01-11T18:42:00.412882Z	info	FLAG: --log_stacktrace_level="default:none"
2024-01-11T18:42:00.412893Z	info	FLAG: --log_target="[stdout]"
2024-01-11T18:42:00.412900Z	info	FLAG: --meshConfig="./etc/istio/config/mesh"
2024-01-11T18:42:00.412904Z	info	FLAG: --outlierLogPath=""
2024-01-11T18:42:00.412908Z	info	FLAG: --proxyComponentLogLevel=""
2024-01-11T18:42:00.412913Z	info	FLAG: --proxyLogLevel="warning,misc:error"
2024-01-11T18:42:00.412918Z	info	FLAG: --serviceCluster="istio-proxy"
2024-01-11T18:42:00.412928Z	info	FLAG: --stsPort="0"
2024-01-11T18:42:00.412932Z	info	FLAG: --templateFile=""
2024-01-11T18:42:00.412938Z	info	FLAG: --tokenManagerPlugin="GoogleTokenExchange"
2024-01-11T18:42:00.412948Z	info	FLAG: --vklog="0"
2024-01-11T18:42:00.412953Z	info	Version 1.16.1-f6d7bf648e571a6a523210d97bde8b489250354b-Clean
2024-01-11T18:42:00.414107Z	info	Maximum file descriptors (ulimit -n): 1024
2024-01-11T18:42:00.414242Z	info	Proxy role	ips=[10.10.10.10] type=sidecar id=ip-10-10-10-10.vm-mysqldb domain=vm-mysqldb.svc.cluster.local
2024-01-11T18:42:00.414290Z	info	Apply mesh config from file defaultConfig:
  discoveryAddress: istiod.istio-system.svc:15012
  holdApplicationUntilProxyStarts: true
  proxyMetadata:
    CANONICAL_REVISION: latest
    CANONICAL_SERVICE: vm-mysqldb
    ISTIO_META_AUTO_REGISTER_GROUP: vm-mysqldb
    ISTIO_META_CLUSTER_ID: Kubernetes
    ISTIO_META_DNS_CAPTURE: "true"
    ISTIO_META_MESH_ID: ""
    ISTIO_META_NETWORK: ""
    ISTIO_META_WORKLOAD_NAME: vm-mysqldb
    ISTIO_METAJSON_LABELS: '{"app":"vm-mysqldb","service.istio.io/canonical-name":"vm-mysqldb","service.istio.io/canonical-revision":"latest"}'
    POD_NAMESPACE: vm-mysqldb
    SERVICE_ACCOUNT: vm-mysqldb
    TRUST_DOMAIN: cluster.local
  terminationDrainDuration: 30s

2024-01-11T18:42:00.416075Z	info	Apply proxy config from env
serviceCluster: vm-mysqldb.vm-mysqldb
controlPlaneAuthPolicy: MUTUAL_TLS

2024-01-11T18:42:00.416630Z	info	Effective config: binaryPath: /usr/local/bin/envoy
concurrency: 2
configPath: ./etc/istio/proxy
controlPlaneAuthPolicy: MUTUAL_TLS
discoveryAddress: istiod.istio-system.svc:15012
drainDuration: 45s
holdApplicationUntilProxyStarts: true
parentShutdownDuration: 60s
proxyAdminPort: 15000
proxyMetadata:
  CANONICAL_REVISION: latest
  CANONICAL_SERVICE: vm-mysqldb
  ISTIO_META_AUTO_REGISTER_GROUP: vm-mysqldb
  ISTIO_META_CLUSTER_ID: Kubernetes
  ISTIO_META_DNS_CAPTURE: "true"
  ISTIO_META_MESH_ID: ""
  ISTIO_META_NETWORK: ""
  ISTIO_META_WORKLOAD_NAME: vm-mysqldb
  ISTIO_METAJSON_LABELS: '{"app":"vm-mysqldb","service.istio.io/canonical-name":"vm-mysqldb","service.istio.io/canonical-revision":"latest"}'
  POD_NAMESPACE: vm-mysqldb
  SERVICE_ACCOUNT: vm-mysqldb
  TRUST_DOMAIN: cluster.local
serviceCluster: vm-mysqldb.vm-mysqldb
statNameLength: 189
statusPort: 15020
terminationDrainDuration: 30s
tracing:
  zipkin:
    address: zipkin.istio-system:9411

2024-01-11T18:42:00.416651Z	info	JWT policy is third-party-jwt
2024-01-11T18:42:00.416657Z	info	using credential fetcher of JWT type in cluster.local trust domain
2024-01-11T18:42:00.417947Z	info	Opening status port 15020
2024-01-11T18:42:00.418004Z	info	dns	Starting local udp DNS server on 127.0.0.1:15053
2024-01-11T18:42:00.418065Z	info	dns	Starting local tcp DNS server on 127.0.0.1:15053
2024-01-11T18:42:00.418480Z	info	Workload SDS socket not found. Starting Istio SDS Server
2024-01-11T18:42:00.418498Z	info	CA Endpoint istiod.istio-system.svc:15012, provider Citadel
2024-01-11T18:42:00.418508Z	info	Using CA istiod.istio-system.svc:15012 cert with certs: ./etc/certs/root-cert.pem
2024-01-11T18:42:00.418588Z	info	citadelclient	Citadel client using custom root cert: ./etc/certs/root-cert.pem
2024-01-11T18:42:00.435960Z	info	ads	All caches have been synced up in 24.85206ms, marking server ready
2024-01-11T18:42:00.436245Z	info	xdsproxy	Initializing with upstream address "istiod.istio-system.svc:15012" and cluster "Kubernetes"
2024-01-11T18:42:00.437929Z	info	Pilot SAN: [istiod.istio-system.svc]
2024-01-11T18:42:00.459561Z	info	sds	Starting SDS grpc server
2024-01-11T18:42:00.459723Z	info	starting Http service at 127.0.0.1:15004
2024-01-11T18:42:00.459731Z	info	Starting proxy agent
2024-01-11T18:42:00.459801Z	info	starting
2024-01-11T18:42:00.459860Z	info	Envoy command: [-c etc/istio/proxy/envoy-rev.json --drain-time-s 45 --drain-strategy immediate --parent-shutdown-time-s 60 --local-address-ip-version v4 --file-flush-interval-msec 1000 --disable-hot-restart --log-format %Y-%m-%dT%T.%fZ	%l	envoy %n	%v -l warning --component-log-level misc:error --concurrency 2]
2024-01-11T18:42:00.612029Z	info	cache	generated new workload certificate	latency=171.806201ms ttl=23h59m59.387981926s
2024-01-11T18:42:00.612077Z	info	cache	Root cert has changed, start rotating root cert
2024-01-11T18:42:00.612102Z	info	ads	XDS: Incremental Pushing:0 ConnectedEndpoints:0 Version:
2024-01-11T18:42:00.612554Z	info	cache	returned workload trust anchor from cache	ttl=23h59m59.387451322s
2024-01-11T18:42:00.615161Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2024-01-11T18:42:00.646047Z	info	ads	ADS: new connection for node:ip-10-10-10-10.vm-mysqldb-1
2024-01-11T18:42:00.646138Z	info	cache	returned workload trust anchor from cache	ttl=23h59m59.353866864s
2024-01-11T18:42:00.646365Z	info	ads	SDS: PUSH request for node:ip-10-10-10-10.vm-mysqldb resources:1 size:924B resource:ROOTCA
2024-01-11T18:42:00.647065Z	info	ads	ADS: new connection for node:ip-10-10-10-10.vm-mysqldb-2
2024-01-11T18:42:00.647141Z	info	cache	returned workload certificate from cache	ttl=23h59m59.352862206s
2024-01-11T18:42:00.647261Z	info	ads	SDS: PUSH request for node:ip-10-10-10-10.vm-mysqldb resources:1 size:5.7kB resource:default

# Deploy the httpbin service on the default namespace:
$ kubectl apply -n default -f samples/httpbin/httpbin.yaml

# Perform a request to the httpbin service on Kubernetes from the Virtual Machine instance to validate that works.
$ root@ip-10-10-10-10:~# curl -vvv httpbin.istio-system.svc.cluster.local:8000/headers
* Expire in 0 ms for 6 (transfer 0x55d6d483afb0)
* Expire in 1 ms for 1 (transfer 0x55d6d483afb0)
* Expire in 0 ms for 1 (transfer 0x55d6d483afb0)
* Expire in 1 ms for 1 (transfer 0x55d6d483afb0)
* Expire in 0 ms for 1 (transfer 0x55d6d483afb0)
* Expire in 0 ms for 1 (transfer 0x55d6d483afb0)
* Expire in 1 ms for 1 (transfer 0x55d6d483afb0)
* Expire in 0 ms for 1 (transfer 0x55d6d483afb0)
* Expire in 0 ms for 1 (transfer 0x55d6d483afb0)
* Expire in 1 ms for 1 (transfer 0x55d6d483afb0)
* Expire in 0 ms for 1 (transfer 0x55d6d483afb0)
* Expire in 0 ms for 1 (transfer 0x55d6d483afb0)
* Expire in 0 ms for 1 (transfer 0x55d6d483afb0)
*   Trying 172.20.89.176...
* TCP_NODELAY set
* Expire in 200 ms for 4 (transfer 0x55d6d483afb0)
* Connected to httpbin.istio-system.svc.cluster.local (172.20.89.176) port 8000 (#0)
> GET /headers HTTP/1.1
> Host: httpbin.istio-system.svc.cluster.local:8000
> User-Agent: curl/7.64.0
> Accept: */*
>
< HTTP/1.1 200 OK
< server: envoy
< date: Thu, 11 Jan 2024 18:45:05 GMT
< content-type: application/json
< content-length: 1019
< access-control-allow-origin: *
< access-control-allow-credentials: true
< x-envoy-upstream-service-time: 1
<
{
  "headers": {
    "Accept": "*/*",
    "Host": "httpbin.istio-system.svc.cluster.local:8000",
    "User-Agent": "curl/7.64.0",
    "X-B3-Sampled": "0",
    "X-B3-Spanid": "xxxxxxxx",
    "X-B3-Traceid": "xxxxxxxx",
    "X-Envoy-Attempt-Count": "1",
    "X-Envoy-Decorator-Operation": "httpbin.istio-system.svc.cluster.local:8000/*",
    "X-Envoy-Peer-Metadata": "xxxxxxxx",
    "X-Envoy-Peer-Metadata-Id": "sidecar~10.10.10.10~ip-10-10-10-10.vm-mysqldb~vm-mysqldb.svc.cluster.local"
  }
}
* Connection #0 to host httpbin.istio-system.svc.cluster.local left intact

```

## Troubleshoot configuration issues

[Docs](https://istio.io/latest/docs/ops/common-problems/)

## Common Problems

**Read extreamly carefully the following Istio pages to undertand it properly**

1. [Traffic Management Problems](https://istio.io/latest/docs/ops/common-problems/network-issues/)
2. [Sidecar Injection Problems](https://istio.io/latest/docs/ops/common-problems/injection/)
3. [Security Problems](https://istio.io/latest/docs/ops/common-problems/security-issues/)
4. [Configuration Validation Problems](https://istio.io/latest/docs/ops/common-problems/validation/)
5. [Observability Problems](https://istio.io/latest/docs/ops/common-problems/observability-issues/)
6. [Upgrade Problems](https://istio.io/latest/docs/ops/common-problems/upgrade-issues/)

## Diagnostic Tools

[Docs](https://istio.io/latest/docs/ops/diagnostic-tools/)

1. [Using the Istioctl Command-line Tool](https://istio.io/latest/docs/ops/diagnostic-tools/istioctl/)
2. [Diagnose your Configuration with Istioctl Analyze](https://istio.io/latest/docs/ops/diagnostic-tools/istioctl-analyze/)
3. [Component Logging](https://istio.io/latest/docs/ops/diagnostic-tools/component-logging/)
4. [Troubleshooting the Istio CNI plugin](https://istio.io/latest/docs/ops/diagnostic-tools/cni/)
5. [Debugging Envoy and Istiod](https://istio.io/latest/docs/ops/diagnostic-tools/proxy-cmd/)
6. [Verifying Istio Sidecar Injection with Istioctl Check-Inject](https://istio.io/latest/docs/ops/diagnostic-tools/check-inject/)
7. [Debugging Virtual Machines](https://istio.io/latest/docs/ops/diagnostic-tools/virtual-machines/)
8. [Understand your Mesh with Istioctl Describe](https://istio.io/latest/docs/ops/diagnostic-tools/istioctl-describe/)
9. [Istiod Introspection](https://istio.io/latest/docs/ops/diagnostic-tools/controlz/)
10. [Troubleshooting Multicluster](https://istio.io/latest/docs/ops/diagnostic-tools/multicluster/)
