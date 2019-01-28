# Kubernetes cluster-api-provider-kubemark Project

This repository hosts an implementation of a provider for Kubemark for the [cluster-api project](https://sigs.k8s.io/cluster-api).

## Community, discussion, contribution, and support

Learn how to engage with the Kubernetes community on the [community page](http://kubernetes.io/community/).

You can reach the maintainers of this project at:

- [#cluster-api on Kubernetes Slack](http://slack.k8s.io/messages/cluster-api)
- [SIG-Cluster-Lifecycle Mailing List](https://groups.google.com/forum/#!forum/kubernetes-sig-cluster-lifecycle)

### Code of conduct

Participation in the Kubernetes community is governed by the [Kubernetes Code of Conduct](code-of-conduct.md).

### How to build the images in the RH infrastructure
The Dockerfiles use `as builder` in the `FROM` instruction which is not currently supported
by the RH's docker fork (see [https://github.com/kubernetes-sigs/kubebuilder/issues/268](https://github.com/kubernetes-sigs/kubebuilder/issues/268)).
One needs to run the `imagebuilder` command instead of the `docker build`.

Note: this info is RH only, it needs to be backported every time the `README.md` is synced with the upstream one.

## How to deploy and test the machine controller with minikube

1. **Install kvm**

    Depending on your virtualization manager you can choose a different [driver](https://github.com/kubernetes/minikube/blob/master/docs/drivers.md).
    In order to install kvm, you can run (as described in the [drivers](https://github.com/kubernetes/minikube/blob/master/docs/drivers.md#kvm2-driver) documentation):

    ```sh
    $ sudo yum install libvirt-daemon-kvm qemu-kvm libvirt-daemon-config-network
    $ systemctl start libvirtd
    $ sudo usermod -a -G libvirt $(whoami)
    $ newgrp libvirt
    ```

    To install to kvm2 driver:

    ```sh
    curl -Lo docker-machine-driver-kvm2 https://storage.googleapis.com/minikube/releases/latest/docker-machine-driver-kvm2 \
    && chmod +x docker-machine-driver-kvm2 \
    && sudo cp docker-machine-driver-kvm2 /usr/local/bin/ \
    && rm docker-machine-driver-kvm2
    ```

2. **Deploying the cluster**

    To install minikube `v0.30.0`, you can run:

    ```sg
    $ curl -Lo minikube https://storage.googleapis.com/minikube/releases/v0.30.0/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/
    ```

    To deploy the cluster:

    ```
    # minikube start --vm-driver kvm2 --extra-config=apiserver.enable-admission-plugins=Initializers,NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota --kubernetes-version v1.11.3 --v 5
    ```

    The minikube enables `NodeRestriction` admission plugin which does not allow to use the same minikube kubeconfig for multiple kubelets
    and to send create requests from static (mirror) nodes.

3. **Create configmap with kubemark kubeconfig**

    Assuming the kubeconfig file is under `~/.kube/kubelet.conf`:

    ```
    $ kubectl create secret generic "kubeconfig" --type=Opaque --from-literal=kubelet.kubeconfig="$(cat ~/.kube/kubelet.conf)"
    ```

    The kubeconfig file can be collected from the minikube by running:

    ```
    $ sudo minikube ssh 'sudo cat /etc/kubernetes/kubelet.conf' > ~/.kube/kubelet.conf
    ```

    Don't forget to update the `server` to point to reachable ip address (e.g. `https://192.168.39.60:8443`).
    Otherwise, the config sets the field to `localhost` which is not accessible from running kubemark.

4. **Deploying the cluster-api stack manifests**

    ``` sh
    $ cd config/default && kustomize build | kubectl apply -f -
    ```

## Provider config

Currently the kubemark actuator allows to configure the following test scenarios:

- have a node report `Unready` status (e.g. for 5s), then Ready (e.g. for the next 40s) and again (periodically):
  ```yaml
  apiVersion: kubemarkproviderconfig.k8s.io/v1alpha1
  kind: KubemarkMachineProviderConfig
  unhealthyDuration: 5s
  healthyDuration: 40s
  turnUnhealthyPeriodically: true
  ```

- have a node report `Unready` status (e.g. 40s after kubelet startup) indefinitely:
  ```yaml
  apiVersion: kubemarkproviderconfig.k8s.io/v1alpha1
  kind: KubemarkMachineProviderConfig
  unhealthyDuration: 40s
  turnUnhealthyAfter: true
  ```

## Kubemark

The provided kubemark (through `gofed/kubemark-machine-controllers:d4f6edb`) is slightly updated version of the kubemark.
The list of PRs that allow kubemark to force kubelet to have node go Unready and/or back.

**Upstream PRs**:
- Setting ProviderID when running Kubemark: https://github.com/kubernetes/kubernetes/pull/73393
- Injecting external Kubelet runtime health checker to simulate node failures:  https://github.com/kubernetes/kubernetes/pull/73398
- Allowing Kubemark to conditionally distrupt Kubelet runtime: https://github.com/kubernetes/kubernetes/pull/73399

**How to build the kubemark image**

1. Clone `k8s.io/kubernetes` repo under `$GOPATH/src/k8s.io/kubernetes`
1. Checkout to required version (e.g. `$ git checkout v1.11.3`)
1. Apply the PRs (and rebase if needed)
1. Run `make WHAT="cmd/kubemark"`
1. `$ cp _output/bin/kubemark cluster/images/kubemark/`
1. `cd cluster/images/kubemark/`
1. Build docker image: `make build REGISTRY=... IMAGE_TAG=...`
1. Push the docker image to registry: `docker push $REGISTRY/`
