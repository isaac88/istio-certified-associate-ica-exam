# Istio Certified Associate (ICA)

Istio Certified Associate exam preparation repo

https://training.linuxfoundation.org/certification/istio-certified-associate-ica/

## Domains & Competencies: Practice exercices

## Pre-Requirements

- Local K8S Kubernetes environment
```bash
$ colima version
colima version 0.6.6
git commit: 9ed7f4337861931b4d0192ca5409683a4b7d1cdc

runtime: docker
arch: aarch64
client: v24.0.7
server: v24.0.7

# Start K3S cluster using Colima
$ colima start --kubernetes --cpu 4 --memory 8

$ kubectl version
Client Version: v1.28.4
Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3
Server Version: v1.28.3+k3s2
```

- Istio CLI 1.18.2 installed 

*[The ICA environment is currently running Istio 1.18.2](https://docs.linuxfoundation.org/tc-docs/certification/important-instructions-ica#ica-exam-environment)*

```bash
# Download Istio for you specific OS and ARCH for the version 1.18.2
$ curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.18.2 sh -
$ ./istio-1.18.2/bin/istioctl version -s --remote=false
1.18.2
``````
