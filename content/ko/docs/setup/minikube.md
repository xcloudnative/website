---
title: Minikube로 로컬 상에서 쿠버네티스 구동
---

Minikube is a tool that makes it easy to run Kubernetes locally. Minikube runs a single-node Kubernetes cluster inside a VM on your laptop for users looking to try out Kubernetes or develop with it day-to-day.

{{< toc >}}

### Minikube 특징

* Minikube supports Kubernetes features such as:
  * DNS
  * NodePorts
  * ConfigMaps and Secrets
  * Dashboards
  * Container Runtime: Docker, [rkt](https://github.com/rkt/rkt), [CRI-O](https://github.com/kubernetes-incubator/cri-o) and [containerd](https://github.com/containerd/containerd)
  * Enabling CNI (Container Network Interface)
  * Ingress

## 설치

See [Installing Minikube](/docs/tasks/tools/install-minikube/).

## 빠른 시작

Here's a brief demo of minikube usage.
If you want to change the VM driver add the appropriate `--vm-driver=xxx` flag to `minikube start`. Minikube supports
the following drivers:

* virtualbox
* vmwarefusion
* kvm2 ([driver installation](https://git.k8s.io/minikube/docs/drivers.md#kvm2-driver))
* kvm ([driver installation](https://git.k8s.io/minikube/docs/drivers.md#kvm-driver))
* hyperkit ([driver installation](https://git.k8s.io/minikube/docs/drivers.md#hyperkit-driver))
* xhyve ([driver installation](https://git.k8s.io/minikube/docs/drivers.md#xhyve-driver)) (deprecated)

Note that the IP below is dynamic and can change. It can be retrieved with `minikube ip`.

```shell
$ minikube start
Starting local Kubernetes cluster...
Running pre-create checks...
Creating machine...
Starting local Kubernetes cluster...

$ kubectl run hello-minikube --image=k8s.gcr.io/echoserver:1.10 --port=8080
deployment.apps/hello-minikube created
$ kubectl expose deployment hello-minikube --type=NodePort
service/hello-minikube exposed

# We have now launched an echoserver pod but we have to wait until the pod is up before curling/accessing it
# via the exposed service.
# To check whether the pod is up and running we can use the following:
$ kubectl get pod
NAME                              READY     STATUS              RESTARTS   AGE
hello-minikube-3383150820-vctvh   0/1       ContainerCreating   0          3s
# We can see that the pod is still being created from the ContainerCreating status
$ kubectl get pod
NAME                              READY     STATUS    RESTARTS   AGE
hello-minikube-3383150820-vctvh   1/1       Running   0          13s
# We can see that the pod is now Running and we will now be able to curl it:
$ curl $(minikube service hello-minikube --url)


Hostname: hello-minikube-7c77b68cff-8wdzq

Pod Information:
	-no pod information available-

Server values:
	server_version=nginx: 1.13.3 - lua: 10008

Request Information:
	client_address=172.17.0.1
	method=GET
	real path=/
	query=
	request_version=1.1
	request_scheme=http
	request_uri=http://192.168.99.100:8080/

Request Headers:
	accept=*/*
	host=192.168.99.100:30674
	user-agent=curl/7.47.0

Request Body:
	-no body in request-


$ kubectl delete services hello-minikube
service "hello-minikube" deleted
$ kubectl delete deployment hello-minikube
deployment.extensions "hello-minikube" deleted
$ minikube stop
Stopping local Kubernetes cluster...
Stopping "minikube"...
```

### 다른 컨테이너 런타임

#### containerd

To use [containerd](https://github.com/containerd/containerd) as the container runtime, run:

```bash
$ minikube start \
    --network-plugin=cni \
    --container-runtime=containerd \
    --bootstrapper=kubeadm
```

Or you can use the extended version:

```bash
$ minikube start \
    --network-plugin=cni \
    --extra-config=kubelet.container-runtime=remote \
    --extra-config=kubelet.container-runtime-endpoint=unix:///run/containerd/containerd.sock \
    --extra-config=kubelet.image-service-endpoint=unix:///run/containerd/containerd.sock \
    --bootstrapper=kubeadm
```

#### CRI-O

To use [CRI-O](https://github.com/kubernetes-incubator/cri-o) as the container runtime, run:

```bash
$ minikube start \
    --network-plugin=cni \
    --container-runtime=cri-o \
    --bootstrapper=kubeadm
```

Or you can use the extended version:

```bash
$ minikube start \
    --network-plugin=cni \
    --extra-config=kubelet.container-runtime=remote \
    --extra-config=kubelet.container-runtime-endpoint=/var/run/crio.sock \
    --extra-config=kubelet.image-service-endpoint=/var/run/crio.sock \
    --bootstrapper=kubeadm
```

#### rkt 컨테이너 엔진

To use [rkt](https://github.com/rkt/rkt) as the container runtime run:

```shell
$ minikube start \
    --network-plugin=cni \
    --container-runtime=rkt
```

This will use an alternative minikube ISO image containing both rkt, and Docker, and enable CNI networking.

### 드라이버 플러그인

See [DRIVERS](https://git.k8s.io/minikube/docs/drivers.md) for details on supported drivers and how to install
plugins, if required.

### 도커 데몬 재사용

When using a single VM of Kubernetes, it's really handy to reuse the minikube's built-in Docker daemon; as this means you don't have to build a docker registry on your host machine and push the image into it - you can just build inside the same docker daemon as minikube which speeds up local experiments. Just make sure you tag your Docker image with something other than 'latest' and use that tag while you pull the image. Otherwise, if you do not specify version of your image, it will be assumed as `:latest`, with pull image policy of `Always` correspondingly, which may eventually result in `ErrImagePull` as you may not have any versions of your Docker image out there in the default docker registry (usually DockerHub) yet.

To be able to work with the docker daemon on your mac/linux host use the `docker-env command` in your shell:

```
eval $(minikube docker-env)
```
You should now be able to use docker on the command line on your host mac/linux machine talking to the docker daemon inside the minikube VM:

```
docker ps
```

On Centos 7, docker may report the following error:

```
Could not read CA certificate "/etc/docker/ca.pem": open /etc/docker/ca.pem: no such file or directory
```

The fix is to update /etc/sysconfig/docker to ensure that minikube's environment changes are respected:

```
< DOCKER_CERT_PATH=/etc/docker
---
> if [ -z "${DOCKER_CERT_PATH}" ]; then
>   DOCKER_CERT_PATH=/etc/docker
> fi
```

Remember to turn off the imagePullPolicy:Always, as otherwise Kubernetes won't use images you built locally.

## 클러스터 관리

### 클러스터 시작

The `minikube start` command can be used to start your cluster.
This command creates and configures a virtual machine that runs a single-node Kubernetes cluster.
This command also configures your [kubectl](/docs/user-guide/kubectl-overview/) installation to communicate with this cluster.

If you are behind a web proxy, you will need to pass this information in e.g. via

```
https_proxy=<my proxy> minikube start --docker-env http_proxy=<my proxy> --docker-env https_proxy=<my proxy> --docker-env no_proxy=192.168.99.0/24
```

Unfortunately just setting the environment variables will not work.

Minikube will also create a "minikube" context, and set it to default in kubectl.
To switch back to this context later, run this command: `kubectl config use-context minikube`.

#### 쿠버네티스 버전 지정

You can specify the specific version of Kubernetes for Minikube to use by
adding the `--kubernetes-version` string to the `minikube start` command. For
example, to run version `v1.7.3`, you would run the following:

```
minikube start --kubernetes-version v1.7.3
```

### 쿠버네티스 구성

Minikube has a "configurator" feature that allows users to configure the Kubernetes components with arbitrary values.
To use this feature, you can use the `--extra-config` flag on the `minikube start` command.

This flag is repeated, so you can pass it several times with several different values to set multiple options.

This flag takes a string of the form `component.key=value`, where `component` is one of the strings from the below list, `key` is a value on the
configuration struct and `value` is the value to set.

Valid keys can be found by examining the documentation for the Kubernetes `componentconfigs` for each component.
Here is the documentation for each supported configuration:

* [kubelet](https://godoc.org/k8s.io/kubernetes/pkg/kubelet/apis/kubeletconfig#KubeletConfiguration)
* [apiserver](https://godoc.org/k8s.io/kubernetes/cmd/kube-apiserver/app/options#ServerRunOptions)
* [proxy](https://godoc.org/k8s.io/kubernetes/pkg/proxy/apis/kubeproxyconfig#KubeProxyConfiguration)
* [controller-manager](https://godoc.org/k8s.io/kubernetes/pkg/apis/componentconfig#KubeControllerManagerConfiguration)
* [etcd](https://godoc.org/github.com/coreos/etcd/etcdserver#ServerConfig)
* [scheduler](https://godoc.org/k8s.io/kubernetes/pkg/apis/componentconfig#KubeSchedulerConfiguration)

#### 예제

To change the `MaxPods` setting to 5 on the Kubelet, pass this flag: `--extra-config=kubelet.MaxPods=5`.

This feature also supports nested structs. To change the `LeaderElection.LeaderElect` setting to `true` on the scheduler, pass this flag: `--extra-config=scheduler.LeaderElection.LeaderElect=true`.

To set the `AuthorizationMode` on the `apiserver` to `RBAC`, you can use: `--extra-config=apiserver.Authorization.Mode=RBAC`.

### 클러스터 중지
The `minikube stop` command can be used to stop your cluster.
This command shuts down the minikube virtual machine, but preserves all cluster state and data.
Starting the cluster again will restore it to it's previous state.

### 클러스터 삭제
The `minikube delete` command can be used to delete your cluster.
This command shuts down and deletes the minikube virtual machine. No data or state is preserved.

## 클러스터와 상호 작용

### Kubectl

The `minikube start` command creates a "[kubectl context](/docs/reference/generated/kubectl/kubectl-commands/#-em-set-context-em-)" called "minikube".
This context contains the configuration to communicate with your minikube cluster.

Minikube sets this context to default automatically, but if you need to switch back to it in the future, run:

`kubectl config use-context minikube`,

Or pass the context on each command like this: `kubectl get pods --context=minikube`.

### 대시보드

To access the [Kubernetes Dashboard](/docs/tasks/access-application-cluster/web-ui-dashboard/), run this command in a shell after starting minikube to get the address:

```shell
minikube dashboard
```

### 서비스

To access a service exposed via a node port, run this command in a shell after starting minikube to get the address:

```shell
minikube service [-n NAMESPACE] [--url] NAME
```

## 네트워킹

The minikube VM is exposed to the host system via a host-only IP address, that can be obtained with the `minikube ip` command.
Any services of type `NodePort` can be accessed over that IP address, on the NodePort.

To determine the NodePort for your service, you can use a `kubectl` command like this:

`kubectl get service $SERVICE --output='jsonpath="{.spec.ports[0].nodePort}"'`

## 퍼시스턴트 볼륨
Minikube supports [PersistentVolumes](/docs/concepts/storage/persistent-volumes/) of type `hostPath`.
These PersistentVolumes are mapped to a directory inside the minikube VM.

The Minikube VM boots into a tmpfs, so most directories will not be persisted across reboots (`minikube stop`).
However, Minikube is configured to persist files stored under the following host directories:

* `/data`
* `/var/lib/localkube`
* `/var/lib/docker`

Here is an example PersistentVolume config to persist data in the `/data` directory:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0001
spec:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 5Gi
  hostPath:
    path: /data/pv0001/
```

## 호스트 폴더 마운트
Some drivers will mount a host folder within the VM so that you can easily share files between the VM and host.  These are not configurable at the moment and different for the driver and OS you are using.

**Note:** Host folder sharing is not implemented in the KVM driver yet.

| Driver | OS | HostFolder | VM |
| --- | --- | --- | --- |
| VirtualBox | Linux | /home | /hosthome |
| VirtualBox | macOS | /Users | /Users |
| VirtualBox | Windows | C://Users | /c/Users |
| VMware Fusion | macOS | /Users | /Users |
| Xhyve | macOS | /Users | /Users |


## 프라이빗 컨테이너 레지스트리

To access a private container registry, follow the steps on [this page](/docs/concepts/containers/images/).

We recommend you use `ImagePullSecrets`, but if you would like to configure access on the minikube VM you can place the `.dockercfg` in the `/home/docker` directory or the `config.json` in the `/home/docker/.docker` directory.

## 애드온

In order to have minikube properly start or restart custom addons,
place the addons you wish to be launched with minikube in the `~/.minikube/addons`
directory. Addons in this folder will be moved to the minikube VM and
launched each time minikube is started or restarted.

## HTTP 프록시 환경에서 Minikube 사용

Minikube creates a Virtual Machine that includes Kubernetes and a Docker daemon.
When Kubernetes attempts to schedule containers using Docker, the Docker daemon may require external network access to pull containers.

If you are behind an HTTP proxy, you may need to supply Docker with the proxy settings.
To do this, pass the required environment variables as flags during `minikube start`.

For example:

```shell
$ minikube start --docker-env http_proxy=http://$YOURPROXY:PORT \
                 --docker-env https_proxy=https://$YOURPROXY:PORT
```

If your Virtual Machine address is 192.168.99.100, then chances are your proxy settings will prevent kubectl from directly reaching it.
To by-pass proxy configuration for this IP address, you should modify your no_proxy settings. You can do so with:

```shell
$ export no_proxy=$no_proxy,$(minikube ip)
```

## 알려진 이슈
* Features that require a Cloud Provider will not work in Minikube. These include:
  * LoadBalancers
* Features that require multiple nodes. These include:
  * Advanced scheduling policies

## 설계

Minikube uses [libmachine](https://github.com/docker/machine/tree/master/libmachine) for provisioning VMs, and [localkube](https://git.k8s.io/minikube/pkg/localkube) (originally written and donated to this project by [RedSpread](https://github.com/redspread)) for running the cluster.

For more information about minikube, see the [proposal](https://git.k8s.io/community/contributors/design-proposals/cluster-lifecycle/local-cluster-ux.md).

## 추가적인 링크:
* **Goals and Non-Goals**: For the goals and non-goals of the minikube project, please see our [roadmap](https://git.k8s.io/minikube/docs/contributors/roadmap.md).
* **Development Guide**: See [CONTRIBUTING.md](https://git.k8s.io/minikube/CONTRIBUTING.md) for an overview of how to send pull requests.
* **Building Minikube**: For instructions on how to build/test minikube from source, see the [build guide](https://git.k8s.io/minikube/docs/contributors/build_guide.md)
* **Adding a New Dependency**: For instructions on how to add a new dependency to minikube see the [adding dependencies guide](https://git.k8s.io/minikube/docs/contributors/adding_a_dependency.md)
* **Adding a New Addon**: For instruction on how to add a new addon for minikube see the [adding an addon guide](https://git.k8s.io/minikube/docs/contributors/adding_an_addon.md)
* **Updating Kubernetes**: For instructions on how to update kubernetes see the [updating Kubernetes guide](https://git.k8s.io/minikube/docs/contributors/updating_kubernetes.md)

## 커뮤니티

Contributions, questions, and comments are all welcomed and encouraged! minikube developers hang out on [Slack](https://kubernetes.slack.com) in the #minikube channel (get an invitation [here](http://slack.kubernetes.io/)). We also have the [kubernetes-dev Google Groups mailing list](https://groups.google.com/forum/#!forum/kubernetes-dev). If you are posting to the list please prefix your subject with "minikube: ".
