# Istio Certified Associate (ICA)

Istio Certified Associate exam preparation repository.

This repository is created to help me and anyone interested to prepare the Istio Certified Associate (ICA) exam. 

https://training.linuxfoundation.org/blog/istio-certified-associate-ica/
https://training.linuxfoundation.org/certification/istio-certified-associate-ica/

## Domains & Competencies: [Preparation Exam Material]

* [Istio Installation, Upgrade & Configuration 7%](./1_istio_installation_upgrade_configuration/README.md)
* [Traffic Management 40%](./2_traffic_management/README.md)
* [Resilience and Fault Injection 20%](./3_resilience_and_fault_injection/README.md)
* [Securing Workloads 20%](./4_securing_workloads/README.md)
* [Advanced Scenarios 13%](./5_advanced_scenarios/README.md)

## Pre-Requirements

* [Docker Install](https://docs.docker.com/engine/install/)
* [Colima Install](https://github.com/abiosoft/colima?tab=readme-ov-file#installation)
* [yq](https://github.com/kislyuk/yq)
* Local K8S Kubernetes environment up ad running (Colima[K3S])

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

$ yq --version
yq 3.2.2
```

- Istio CLI 1.18.2 installed 

*[The ICA environment is currently running Istio 1.18.2](https://docs.linuxfoundation.org/tc-docs/certification/important-instructions-ica#ica-exam-environment)*

```bash
# Download Istio for you specific OS and ARCH for the version 1.18.2
$ curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.18.2 sh -
$ ./istio-1.18.2/bin/istioctl version -s --remote=false
1.18.2
``````

# External preparation resources

* [Istio official docs](https://istio.io/latest/docs)
* [Istio Service Mesh Workshop](https://www.istioworkshop.io/)
* [Istio 0 to 60 workshop: Video](https://academy.tetrate.io/courses/take/istio-0-to-60/lessons/38552925-watch-the-replay-of-istio-0-to-60-workshop)
* [Istio 0 to 60 workshop: Exercices](https://tetratelabs.github.io/istio-0to60)
* [Mesh week](https://github.com/solo-io/mesh-week)
* [Mesh week: How to prepare for Istio Certified associate exam (ICA)](https://learncloudnative.com/blog/2023-10-10-meshweek)
* [Mesh week: Mock Istio Certified Associate Exam](https://docs.google.com/forms/d/e/1FAIpQLSfD4BLLQfdUwnIyiTBSGC_OzmSbiyrIlNp5Am61fTOhRbfiLw/viewform)
* [Istio Security Problems](https://istio.io/latest/docs/ops/common-problems/security-issues/)
* [Istio Security Best Practices](https://istio.io/latest/docs/ops/best-practices/security/)
* [Life of a packet through Istio by Matt Turner](https://www.youtube.com/watch?v=cB611FtjHcQ)
* [Matt Turner - Life of a Packet through Istio III - WTF is SRE 2023](https://www.youtube.com/watch?v=qsTK4o189_I)
* [Killercoda Istio ICA](https://github.com/killercoda/scenarios-ica)
