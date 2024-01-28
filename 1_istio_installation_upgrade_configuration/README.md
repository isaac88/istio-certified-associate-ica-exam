# Istio Installation, Upgrade & Configuration 7%

* [Using the Istio CLI to install a basic cluster](#using-the-istio-cli-to-install-a-basic-cluster)
* [Customizing the Istio installation with the IstioOperator API](#customizing-the-istio-installation-with-the-istiooperator-api)
* [Using overlays to manage Istio component settings](#using-overlays-to-manage-istio-component-settings)

## Ex1: Using the Istio CLI to install a basic cluster

[Docs](https://istio.io/latest/docs/setup/install/istioctl/)
[Configuration Reference](https://istio.io/latest/docs/reference/config/istio.operator.v1alpha1/)

Requirements:
* Install Istio using the istioctl
* Validate that Istio is properly installed
* Uninstall Istio completly
* Install Istio using the IstioOperator with the demo profile
  * Verify the installation base on the IstioOperator file
  * Check which K8S resources will be created base on that IstioOperator file
  * Compare minimal profile with demo profile

<details>
  <summary>Solution</summary>

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
  $ cat <<EOF >>istiocrd.yaml
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
EOF

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
  ```
</details>

## Ex2: Customizing the Istio installation with the IstioOperator API

[Docs](https://istio.io/latest/docs/setup/additional-setup/customize-installation/)

Requirements:
* Configure IstioOperator demo profile with the following requirements:
  * Enable proxy access logs
  * Create 2 Ingress gateways names: custom-gateway ns custom-ns-name and istio-ingressgateway ns istio-system
  * Disable the Egress gateway
  * Increase the request of IstioD(Pilot) to 500mCPU/500Mib Ram

<details>
  <summary>Solution</summary>

  ```bash
  # The dump subcommand dumps the values in an Istio configuration profile
  $ istioctl profile dump demo -oyaml > istio-demo.yaml
  # We remove from the istio-demo.yaml dump file all the things that we don't need to override.
  $ cat <<EOF>>istiocrd.yaml
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
    pilot:
      # Customize the k8s resources: https://istio.io/latest/docs/reference/config/istio.operator.v1alpha1/#KubernetesResourcesSpec
      k8s:
        resources:
          requests:
            cpu: 500m
            memory: 500Mi
    egressGateways:
    # There can be more than 1 egress gateway - note that to customize a gateway
    # that already exists in the profile (e.g. demo in this case), you need to use the default names
      - name: istio-egressgateway
        enabled: false
    ingressGateways:
        # Disable the default ingress gateway
      - name: istio-ingressgateway
        enabled: true
        # Enable a new ingress gateway
      - name: custom-gateway
        enabled: true
        namespace: custom-ns-name
EOF

  # Create the custom-ns-name ns
  $ kubectl create ns custom-ns-name
  # Install/upgrade Istio using that custom istiocrd.yaml file.
  $ istioctl upgrade -f istiocrd.yaml
  # Check the resources created. 2 Ingress Gateway 1 Istiod
  $  kubectl get pod -A |grep -e istiod -e istio-ingress -e custom-gateway | grep -v svclb
  kubectl get pod -A |grep -e istiod -e istio-ingress -e custom-gateway | grep -v svclb
  istio-system     istiod-57f96db8f6-q6mfm                     1/1     Running   0          18m
  custom-ns-name   custom-gateway-746ddb8dff-8mqbw             1/1     Running   0          18m
  istio-system     istio-ingressgateway-77fbf9d476-8bw8c       1/1     Running   0          17m
  # Check the IstioD has the proper request values set
  $ kubectl get pod -l app=istiod -A  -oyaml |yq '.items[0].spec.containers[]| select(.name == "discovery") | .resources'
  ```
</details>

## Ex3: Using overlays to manage Istio component settings

[Docs](https://istio.io/latest/docs/setup/additional-setup/customize-installation/#patching-the-output-manifest)
[istio.operator.v1alpha1/#K8sObjectOverlay API](https://istio.io/latest/docs/reference/config/istio.operator.v1alpha1/#K8sObjectOverlay)

Requirements:

* Create the following IstioOperator CR patch file:
  * Add a label on the IstioD(Pilot svc): OVERLAY_LABEL: OVERLAY_VALUE
  * Add a label on the IstioD(Pilot Pods): OVERLAY_LABEL: OVERLAY_VALUE
  * Change the --keepaliveMaxServerConnectionAge arg from discovery container to 60m
  * Change the port 8080 from discovery container to 1234
  * Change the POD_NAMESPACE env var from discovery container to fieldPath: metadata.name
  * Delete the REVISION env var from discovery container
  * Delete the securityContext from the discovery container
  * Check the patch file to see if the K8S resources will be patched properly.

<details>
  <summary>Solution</summary>

  ```bash
  # Start from that example in the Istio docs: https://istio.io/latest/docs/setup/additional-setup/customize-installation/#patching-the-output-manifest
  # Modify as required.
  $ cat <<EOF>>patch.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  profile: empty
  components:
    pilot:
      enabled: true
      k8s:
        overlays:
          - kind: Deployment
            name: istiod
            patches:
              # Change the --keepaliveMaxServerConnectionAge arg from discovery container to 60m
              - path: spec.template.spec.containers.[name:discovery].args.[30m]
                value: "60m" # overridden from 30m
              # Change the port 8080 from discovery container to 1234
              - path: spec.template.spec.containers.[name:discovery].ports.[containerPort:8080].containerPort
                value: 1234
              # Change the POD_NAMESPACE env var from discovery container to fieldPath: metadata.name
              - path: spec.template.spec.containers.[name:discovery].env.[name:POD_NAMESPACE].valueFrom
                value: |
                  fieldRef:
                    apiVersion: v2
                    fieldPath: metadata.name
              # Delete the REVISION env var from discovery container
              - path: spec.template.spec.containers.[name:discovery].env.[name:REVISION]
              # Delete the securityContext from the discovery container
              - path: spec.template.spec.containers.[name:discovery].securityContext
              # Add a label on the IstioD(Pilot Pods): OVERLAY_LABEL: OVERLAY_VALUE
              - path: spec.template.metadata.labels.OVERLAY_LABEL
                value: OVERLAY_VALUE
          - kind: Service
            name: istiod
            patches:
              # Add a label on the IstioD(Pilot svc): OVERLAY_LABEL: OVERLAY_VALUE
              - path: metadata.labels.OVERLAY_LABEL
                value: OVERLAY_VALUE
EOF

  # Check the overlay patch using the generate manifest before to apply it
  # Check that all requirements are meet.
  $ istioctl manifest generate -f patch.yaml
  ```
</details>
