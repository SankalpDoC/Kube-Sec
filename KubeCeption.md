# Kube-Ception

## Table of Contents

- [Introduction](#introduction)
- [Features](#features)
- [Infra Setup](#Infra Setup)


## Introduction

### Overview

- Rapid deployment of nested Kubernetes clusters.
- Predefined configurations for reduced manual intervention.
- Seamless pod-VM communication.
- Maintain network isolation for nested clusters. 


### Motivation

In bigger enterprise scenarios a single Kubernetes cluster with its inbuilt isolation mechanisms is often not sufficient to satisfy compliance and security requirements. More advanced (firewalled) zoning or layered security concepts are tough to reproduce with a single installation. With namespace isolation both privileged access as well as firewalled zones can hardly be implemented without sidestepping security measures.

you could go and set up multiple completely separate (and federated) installations of Kubernetes. However, automating the deployment and management of these clusters would need additional tooling and complex monitoring setups. Further, we wanted to be able to spin clusters up and down on demand, scale them, update them, keep track of which clusters are available, and be able to assign them to organizations and teams flexibly. In fact this setup can be combined with a federation control plane to federate deployments to the clusters over one API endpoint.

## Features

This project focuses on optimizing the deployment and management of Kubernetes clusters locally, independent of cloud providers. It introduces automated provisioning processes that streamline the creation of nested Kubernetes clusters within an on-premises environment. By leveraging local resources and tools such as VM templates and configuration management frameworks like Ansible or Kubernetes itself, the project enables rapid deployment and scaling of clusters as needed.

Key features include:

- Local Provisioning: Utilizing local infrastructure to create and manage Kubernetes clusters, reducing dependency on external cloud services.

- Automated Deployment: Implementing scripts or tools for automated provisioning and configuration of cluster nodes, ensuring consistency and reducing manual intervention.

- Networking and Security: Addressing complex networking configurations and security requirements within the local environment, such as setting up network policies and securing cluster communications.

- Scalability and Flexibility: Supporting dynamic scaling of clusters based on workload demands, with the ability to spin up new clusters quickly using predefined VM templates or container images.

- Monitoring and Maintenance: Implementing monitoring tools and practices to ensure cluster health and performance, along with strategies for ongoing maintenance and updates.

- Integration with Virtualization: Leveraging technologies like KubeVirt or other virtualization solutions to manage VMs alongside containers, enhancing resource utilization and workload flexibility.

- Customization and Adaptability: Providing flexibility to tailor configurations based on specific organizational needs and environments, adapting to different use cases and scenarios.

- Documentation and Knowledge Sharing: Emphasizing comprehensive documentation and knowledge sharing practices to facilitate team collaboration and support ongoing development and operations.

## Infra Setup

### Prerequisites

- Check nested virt support:

  ```bash
  egrep -c '(vmx|svm)' /proc/cpuinfo
  ```
  The value returned should be greater than zero
- Docker Installation:

  ```bash
  # Add Docker's official GPG key:
  sudo apt-get update
  sudo apt-get install ca-certificates curl
  sudo install -m 0755 -d /etc/apt/keyrings
  sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
  sudo chmod a+r /etc/apt/keyrings/docker.asc

  # Add the repository to Apt sources:
  echo \
    "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
    $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
  sudo apt-get update

  sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
  ```



- Deployment commands:

  Just to avoid restart crashes while updation.
  ```bash
  sudo apt-mark hold linux-generic linux-image-generic linux-headers-generic
  ```

- Kubectl Installation:

  Kubernetes provides a command line tool for communicating with a Kubernetes cluster's control plane, using the Kubernetes API.

  reference: https://kubernetes.io/docs/reference/kubectl/

  ```bash
  curl -LO https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl
  sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
  rm kubectl
  ```



- Virtctl Installation:

  KubeVirt provides an additional binary called virtctl for quick access to the serial and graphical ports of a VM and also handle start/stop operations.

  Reference: https://kubevirt.io/quickstart_kind/

  ```bash
  VERSION=$(k get kubevirt.kubevirt.io/kubevirt -n kubevirt -o=jsonpath="{.status.observedKubeVirtVersion}")
  ARCH=$(uname -s | tr A-Z a-z)-$(uname -m | sed 's/x86_64/amd64/') || windows-amd64.exe
  echo ${ARCH}
  curl -L -o virtctl https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/virtctl-${VERSION}-${ARCH}
  chmod +x virtctl
  sudo install virtctl /usr/local/bin
  ```

- KinD(Kubernetes IN Docker):


  kind is a tool for running local Kubernetes clusters using Docker container “nodes”.
  kind was primarily designed for testing Kubernetes itself, but may be used for local development or CI.

  reference: https://kind.sigs.k8s.io/

  - Installation:

    ```bash
    [ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.22.0/kind-linux-amd64
    chmod +x ./kind
    sudo mv ./kind /usr/local/bin/kind
    ```
- Modifying .bashrc 

  ```bash
  complete -o default -F __start_kubectl k
  source <(kubectl completion bash)
  alias k="sudo kubectl"
  alias v="sudo virtctl"
  alias w=”watch sudo kubectl”
  alias c=clear
  complete -F __start_kubectl k
  ```
  Apply these changes to the current terminal
  ```bash
  source ~/.bashrc
  ```

### Deployment Prep:

- Cluster Creation:

  The following command will create a kind cluster comprizing of three docker nodes, One primary and two worker nodes worker and worker2.

  ```bash

  cat <<EOF | sudo kind create cluster --name child --config=-
  kind: Cluster
  apiVersion: kind.x-k8s.io/v1alpha4
  nodes:
    - role: control-plane
      image: kindest/node:v1.29.2@sha256:51a1434a5397193442f0be2a297b488b6c919ce8a3931be0ce822606ea5ca245
    - role: worker
      image: kindest/node:v1.29.2@sha256:51a1434a5397193442f0be2a297b488b6c919ce8a3931be0ce822606ea5ca245
    - role: worker
      image: kindest/node:v1.29.2@sha256:51a1434a5397193442f0be2a297b488b6c919ce8a3931be0ce822606ea5ca245
  EOF

  ```

- ISTIO

  Istio is an open platform-independent service mesh that provides traffic management, policy enforcement, and telemetry collection.

  Open: Istio is being developed and maintained as open-source software. We encourage contributions and feedback from the community at-large.

  Platform-independent: Istio is not targeted at any specific deployment environment. During the initial stages of development, Istio will support Kubernetes-based deployments. However, Istio is being built to enable rapid and easy adaptation to other environments.

  Service mesh: Istio is designed to manage communications between microservices and applications. Without requiring changes to the underlying services, Istio provides automated baseline traffic resilience, service metrics collection, distributed tracing, traffic encryption, protocol upgrades, and advanced routing functionality for all service-to-service communication.

  Reference: https://istio.io/latest/docs/

  - Download
    ```bash
    curl -L https://istio.io/downloadIstio | sh -
    export PATH="$PATH:/home/ubun2/istio-1.22.1/bin"
    sudo install -o root -g root -m 0755 istio-1.22.1/bin/istioctl /usr/local/bin/istioctl
    ```
  - Installation 
    
    To check system readiness do:
    ```
    sudo istioctl x precheck
    ```
    Proceed with installation using:
    ```
    sudo istioctl install --set profile=default
    ```
  - Verification

    Once installed we can verify and also enable istio injection to our default namespace
    ```
    k get all -n istio-system
    k label namespace default istio-injection=enabled
    ```

- KubeVirt: 

  KubeVirt technology addresses the needs of development teams that have adopted or want to adopt Kubernetes but possess existing Virtual Machine-based workloads that cannot be easily containerized. More specifically, the technology provides a unified development platform where developers can build, modify, and deploy applications residing in both Application Containers as well as Virtual Machines in a common, shared environment.

  Reference: https://kubevirt.io/user-guide/


  - Installation
    
    Deploy the KubeVirt, which manages the lifecycle of all the KubeVirt core components.

    ```bash
    export VERSION=$(curl -s https://storage.googleapis.com/kubevirt-prow/release/kubevirt/kubevirt/stable.txt)
    echo $VERSION
    k create -f https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/kubevirt-operator.yaml
    watch sudo kubectl -n kubevirt get all
    ```
    Deploy the KubeVirt custom resource definitions(CRDs):
    ```bash
    k create -f https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/kubevirt-cr.yaml
    watch sudo kubectl -n kubevirt get all
    ```

  - Verification:

    ```bash
    sudo kubectl -n kubevirt get kubevirt
    ```

- Containerized Data Importer(CDI) 

  Containerized-Data-Importer (CDI) is a persistent storage management add-on for Kubernetes. It's primary goal is to provide a declarative way to build Virtual Machine Disks on PVCs for Kubevirt VMs

  CDI works with standard core Kubernetes resources and is storage device agnostic, while its primary focus is to build disk images for Kubevirt, it's also useful outside of a Kubevirt context to use for initializing your Kubernetes Volumes with data.

  Reference: https://github.com/kubevirt/containerized-data-importer/blob/main/README.md

  - Installation
    ```bash
    export VERSION=$(curl -s https://api.github.com/repos/kubevirt/containerized-data-importer/releases/latest | grep '"tag_name":' | sed -E 's/.*"([^"]+)".*/\1/')
    echo $VERSION
    k create -f https://github.com/kubevirt/containerized-data-importer/releases/download/$VERSION/cdi-operator.yaml
    k create -f https://github.com/kubevirt/containerized-data-importer/releases/download/$VERSION/cdi-cr.yaml
    watch sudo kubectl -n cdi get all
    ```
    Expose the “cdi-uploadproxy” Service
    ```bash
    k apply -f - <<EOF 
    apiVersion: v1
    kind: Service
    metadata:
      labels:
        cdi.kubevirt.io: cdi-uploadproxy
      name: cdi-uploadproxy-nodeport
      namespace: cdi
    spec:
      externalTrafficPolicy: Cluster
      internalTrafficPolicy: Cluster
      ipFamilies:
      - IPv4
      ipFamilyPolicy: SingleStack
      ports:
      - nodePort: 31001
        port: 443
        protocol: TCP
        targetPort: 8443
      selector:
        cdi.kubevirt.io: cdi-uploadproxy
      sessionAffinity: None
      type: NodePort
    EOF
    ```


  - DataVolume

    CDI includes a CustomResourceDefinition (CRD) that provides an object of type DataVolume. The DataVolume is an abstraction on top of the standard Kubernetes PVC and can be used to automate creation and population of a PVC with data. Although you can use PVCs directly with CDI, DataVolumes are the preferred method since they offer full functionality, a stable API, and better integration with kubevirt.

    ```bash
    v image-upload dv ubun2-cloud --insecure --access-mode ReadWriteOnce --size 30Gi --image-path ubun-cloud.img --uploadproxy-url https://172.18.0.2:31001 --force-bind
    ```

    Here for the above command, the --uploadproxy-url flag will contain: https://<"NodeIP of the node running the uploadproxy pod">:<"NodePort of the cdi-uploadproxy-nodeport service">

### Deployment

- CloudInit Scripts

  When we deploy a new cloud instance, cloud-init takes an initial configuration that we supply, and it automatically applies those settings when the instance is created. It’s rather like writing a to-do list, and then letting cloud-init deal with that list for us.

  Cloud-init can handle a range of tasks that normally happen when a new instance is created. It’s responsible for activities like setting the hostname, configuring network interfaces, creating user accounts, and even running scripts.

  Reference: https://cloudinit.readthedocs.io/en/latest/index.html


- VM Deployment:

  Apply the VM manifests and make sure there's no error with regards to machine creation and Sidecar injection.

### Monitoring

- Monitoring with Kiali

  Kiali is an observability console for Istio with service mesh configuration and validation capabilities. It helps you understand the structure and health of your service mesh by monitoring traffic flow to infer the topology and report errors. Kiali provides detailed metrics and a basic Grafana integration, which can be used for advanced queries. Distributed tracing is provided by integration with Jaeger.

  Reference: https://istio.io/latest/docs/ops/integrations/kiali/

  - Installation

    Head over to the istio directory we downloaded(curled!) earlier and run: 

    ```
    k create -f samples/addons/kiali.yaml
    k create -f samples/addons/jaeger.yaml
    k create -f samples/addons/prometheus.yaml

    sudo istioctl dashboard kiali
    ```
- Inside LINUX VMs

  To check Ubuntu version: 
  ```
  lsb_release -a
  ```

- Local Path Setup: To enable iso recognition from the filesystem to the cluster
```
k apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.26/deploy/local-path-storage.yaml
k patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
watch sudo kubectl -n local-path-storage get all
```
## Challenges

### Nested Virtualization

  Wasn't able to set up the kube-ception architecture in windows as the nested virtualization option was not enable to any of the hypervisors we seeked(i.e. VirtualBox or VMware). And thus were not able to install kubeVirt's CRDs in the cluster.

  To check for nested virtualization support do: 
  ```
  egrep -c '(vmx|svm)' /proc/cpuinfo
  ```

### Wrong ISO image selection for the VMs

  We need ISO images for the VMs to have an operating system running on them, but we can't just use any .iso as it can lead to underutilization and poor optimization of the resource allocated to the VM, The VMs require a light weight OS which has only the necessary packages with less overhead. For this reason Cloud images are a prefered option. When using the ubuntu desktop ISO we were not able to get network inside the VM and thus we weren't able to get internet connectivity inside it and thus no ssh to the VM was possible.

  Get Ubuntu Cloud Images from here: https://cloud-images.ubuntu.com/

### Unable to inject sidecars to the VMs

  The isito sidecar inject annotation (sidecar.istio.io/inject: "true") should be given to the VMI(Virtual Machine Instance) spec and not in the VM(Virtual Machine) spec. This was the reason of the failure of the istiod to attach a sidecar proxy to the VMs. Also it should be noted that Linkerd can be also used instead of istio. I had no trouble setting up VM workloads to a Linkerd Mesh and in fact found it to be way more straight forward than istio even for setting up Multi-Cluster architecture.

### Issues in setting the Primary-Remote Architecture via Istio

  - Accessing the API server

    For the Primary-Remote to work, both cluster should be having the API Server in each cluster accessible to the other cluster in the mesh. The child cluster being within the VM of the main cluster, made it difficult for the main cluster to reach the API server of the child cluster. I tried to make the API server of the child accessible from the main but merging the kubeconfig of the child cluster.

    ```
    # Copy the child's kubeconfig to the same directory of the main's kubeconfig and run
    export KUBECONFIG=~/.kube/child-config:/path/to/local/config #( Eg. export KUBECONFIG=/root/.kube/child-config:/root/.kube/config )
    k config view --merge --flatten > ~/.kube/merged-config  #You can vim the merged-config and will see the child's contexts,cluster and users inside it.
    mv ~/.kube/merged-config ~/.kube/config # Will replace the config to the merged one.
    ```
    But just having the certificates is not the only thing needed to be able to reach the child cluster. We'll need network connectivity to it as well. And thus while SSHing to the VM, we'll create a tunnel which maps the API server's node port to the Port of the child's API server.
  
    ```
    ssh -L <Child's API server NodePort>:127.0.0.1:<Child's API server NodePort> -p nodePort-VM user@host-machine-ip
    ```

  - Creating Trust Between the Clusters

    For both the clusters to trust each other we need to set up certificates from a common root Certificate Athority(CA) like so 
    ```

    mkdir -p certs
    pushd certs

    sudo make -f ../tools/certs/Makefile.selfsigned.mk root-ca

    sudo make -f ../tools/certs/Makefile.selfsigned.mk main-cacerts
    sudo make -f ../tools/certs/Makefile.selfsigned.mk child-cacerts

    k create ns istio-system
    k create ns istio-system --context kind-child
    
    k create secret generic cacerts -n istio-system  --from-file=main/ca-cert.pem  --from-file=main/ca-key.pem --from-file=main/root-cert.pem  --from-file=main/cert-chain.pem

    k --context kind-child create secret generic cacerts -n istio-system  --from-file=child/ca-cert.pem  --from-file=child/ca-key.pem --from-file=child/root-cert.pem  --from-file=child/cert-chain.pem

    popd
    ```
    Verification
    
    The following shouldn't produce any output if the above steps were done correctly.
    ```
    diff \
    <(k --context=kind-main -n istio-system get secret cacerts -ojsonpath='{.data.root-cert\.pem}') \
    <(k --context=kind-child -n istio-system get secret cacerts -ojsonpath='{.data.root-cert\.pem}')
    ```
