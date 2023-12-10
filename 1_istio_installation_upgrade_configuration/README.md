# Istio Installation, Upgrade & Configuration 7%

* Using the Istio CLI to install a basic cluster
* Customizing the Istio installation with the IstioOperator API
* Using overlays to manage Istio component settings


## Using the Istio CLI to install a basic cluster

[Docs](https://istio.io/latest/docs/setup/install/istioctl/)

```bash

# Verify that Istio can be installed or upgraded
$ istio-1.18.2/bin/istioctl x precheck

# Apply a default Istio installation. Using the "default" profile.
# It'll install istio with the following components included on the "default" profile: "Istio core" "Istiod" "Ingress gateways".
$ istioctl install

# Get PODs running for Istio
$ kubectl get po -n istio-system

# Delete all Istio related sources for all versions
istioctl uninstall --purge

# Install Istio demo profile using istioctl ["Istio core" "Istiod" "Ingress gateways" "Egress gateways"]
$ istioctl install --set profile=demo
$ kubectl get pod -n istio-system |wc -l
       4
$ kubectl get istiooperators.install.istio.io -n istio-system
NAME              REVISION   STATUS   AGE
installed-state                       74s

# Installing Istio demo profile using the IstioOperator CRD ["Istio core" "Istiod" "Ingress gateways" "Egress gateways"]
$ cat > istiocrd.yaml <<EOF
apiVersion: install.istio.io/v1alpha1
# Reference: https://istio.io/latest/docs/reference/config/istio.operator.v1alpha1/
kind: IstioOperator 
spec:
  profile: demo
EOF
$ istioctl install -f istiocrd.yaml
$ kubectl get pod -n istio-system |wc -l
       4
$ kubectl get istiooperators.install.istio.io -n istio-system
NAME              REVISION   STATUS   AGE
installed-state                       74s

# The dump subcommand dumps the values in an Istio configuration profile
# The path the root of the configuration subtree to dump e.g. components.pilot. By default, dump whole tree
# Exam: Compare configurations.
$ istioctl profile dump --config-path components.pilot demo
enabled: true
k8s:
  env:
  - name: PILOT_TRACE_SAMPLING
    value: "100"
  resources:
    requests:
      cpu: 10m
      memory: 100Mi

# The diff subcommand displays the differences between two Istio configuration profiles
$ istioctl profile diff default demo

# The generate subcommand generates an Istio install manifest and outputs to the console by default
$ istioctl manifest generate

# Show differences in manifests
$ istioctl manifest generate > 1.yaml
$ istioctl manifest generate -f samples/operator/pilot-k8s.yaml > 2.yaml

# Verify a successful installation (Passing a generated manifest)
# It'll verify the generated manifest with the current state on the cluster.
$ istioctl manifest generate > generated-manifest.yaml
$ istioctl verify-install -f generated-manifest.yaml

# Verify a successful installation IstioOperator
# If you do not specify an installation it will check for an IstioOperator resource
# and will verify if pods and services defined in it are present.
$ istioctl verify-install

# Helm (Optional)

```


## Customizing the Istio installation with the IstioOperator API

[Docs](https://istio.io/latest/docs/setup/additional-setup/customize-installation/)

```bash

# The dump subcommand dumps the values in an Istio configuration profile
$ istioctl profile dump demo

$ cat > istiocrd.yaml <<EOF
apiVersion: install.istio.io/v1alpha1
# Reference: https://istio.io/latest/docs/reference/config/istio.operator.v1alpha1/
kind: IstioOperator 
spec:
  # Configuration profile used as base for installation
  # Other settings you can configure on this level:
  # namespace, hub, revision, tag
  profile: demo

  # Mesh configuration settings: https://istio.io/latest/docs/reference/config/istio.mesh.v1alpha1/
  meshConfig: 
    accessLogFile: /dev/stdout

  # Individual components you can configure: https://istio.io/latest/docs/reference/config/istio.operator.v1alpha1/#IstioComponentSetSpec
  # Includes base, pilot, cni, ztunnel, ingressGatways, egressGatways, istiodRemote
  components:
    # pilot: ..
    # cni: ...
    egressGateways:
    # There can be more than 1 egress gateway - note that to customize a gateway
    # that already exists in the profile (e.g. demo in this case), you need to use the default names
      - name: istio-egressgateway
        enabled: false
    ingressGateways:
        # Disable the default ingress gateway
      - name: istio-ingressgateway
        enabled: false
        # Enable a new ingress gateway
      - name: peters-gateway
        enabled: true
        namespace: custom-ns-name
        label:
          app: custom-label
        # Customize the k8s resources: https://istio.io/latest/docs/reference/config/istio.operator.v1alpha1/#KubernetesResourcesSpec
        k8s:
          resources:
            requests:
              cpu: 5m
              memory: 20Mi
EOF
```

## Using overlays to manage Istio component settings

[Docs](https://istio.io/latest/docs/setup/additional-setup/customize-installation/#patching-the-output-manifest)
[istio.operator.v1alpha1/#K8sObjectOverlay API](https://istio.io/latest/docs/reference/config/istio.operator.v1alpha1/#K8sObjectOverlay)

```bash

$ cat > patch.yaml <<EOF
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  profile: empty
  hub: docker.io/istio
  tag: 1.1.6
  components:
    pilot:
      enabled: true
      namespace: istio-control
      k8s:
        overlays:
          - kind: Deployment
            name: istiod
            patches:
              # Select list item by value
              - path: spec.template.spec.containers.[name:discovery].args.[30m]
                value: "60m" # overridden from 30m
              # Select list item by key:value
              - path: spec.template.spec.containers.[name:discovery].ports.[containerPort:8080].containerPort
                value: 1234
              # Override with object (note | on value: first line)
              - path: spec.template.spec.containers.[name:discovery].env.[name:POD_NAMESPACE].valueFrom
                value: |
                  fieldRef:
                    apiVersion: v2
                    fieldPath: metadata.myPath
              # Deletion of list item
              - path: spec.template.spec.containers.[name:discovery].env.[name:REVISION]
              # Deletion of map item
              - path: spec.template.spec.containers.[name:discovery].securityContext
          - kind: Service
            name: istiod
            patches:
              - path: spec.ports.[name:https-dns].port
                value: 11111 # OVERRIDDEN
EOF

# Check the overlay patch using the generate manifest before to apply it

$ istioctl manifest generate -f patch.yaml

```