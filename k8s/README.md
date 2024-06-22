# 쿠버네티스

> 이 디렉토리는 [쿠버네티스](https://github.com/kubernetes/kubernetes)의 컴포넌트들의 소스코드를 분석해서 모았습니다.

## 쿠버네티스란?
**"여러 대의 컴퓨터에서 여러 대의 컨테이너를 안정적으로 구동하자"** 는 목적을 골자로 여러 개의 컴포넌트를 통해 가용성을 확보하는 소프트웨어입니다.

### [구성요소](https://kubernetes.io/docs/concepts/overview/components/)
쿠버네티스 클러스터는 Control Plane과 Data Plane으로 나뉘어있습니다. Control Plane은 클러스터 상태의 저장, 그리고 그 상태로 유지하도록 관리하며 그 과정에서 컨테이너의 스케줄링도 맡습니다. 

**Control Plane**
- [`kube-apiserver`](./kube-apiserver/apiserver.md): 쿠버네티스 API를 노출하는 API 서버입니다. 여러 대의 Master Node가 구성되어있으면 여러 API 서버 간의 트래픽을 분산시키면서 작동시킬 수 있습니다.
- `etcd`: key-value 저장소로, 클러스터의 상태를 보관하며, [Raft](https://raft.github.io/)라는 리더선출 알고리즘을 활용해 HA 구성을 지원합니다.
- `kube-scheduler`: Pod(컨테이너의 묶음)를 새로 생성되도록 정의할때 지정된 Node가 없을때, 스케줄링을 해서 지정한 규칙에 따라 배치합니다.
- `kube-controller-manager`: 특정 상태로 유지되도록 loop을 돌면서 확인하는 컴포넌트를 Controller라고 합니다. 예를 들어 Deployment를 배포하면 해당 Pod를 지정한 상태로 계속 유지하도록 loop을 돌면서 확인하며, 원하는 상태에서 벗어났을 때 다시 복구하려고 시도합니다. 이러한 Controller들을 하나의 바이너리로 뭉쳐서 만든게 `kube-controller-manager` 입니다.
- `cloud-controller-manager`: Cloud Provider와 관련된 Controller들을 모은 컴포넌트입니다. 이는 각 Provider에 따라 다르며, 온프레미스로 배포할 경우 존재하지 않습니다.

**Nodes**
해당 컴포넌트들은 Master Node, Worker Node을 가리지 않고 모든 Node에 존재하는 컴포넌트입니다.
- `kubelet`: 각 노드의 컨테이너 상태를 파악하는 컴포넌트입니다. 각 Pod를 정의하는 PodSpec를 가지고 있고, 해당되는 Pod가 정상적으로 작동하는지 확인합니다. 당연하게도 쿠버네티스에서 생성하지 않은 컨테이너는 PodSpec를 가지고 있지 않으므로, 관리하지 않습니다.
- `kube-proxy`: 쿠버네티스 Service 리소스를 위한 Node 내의 네트워크 프록시입니다. 기본적으로 `iptables`를 활용해서 DNAT 정책을 삽입하나, IPVS 등의 다른 방식을 사용할 수도 있으며, 아예 제거해서 CNI에게 역할을 위임할 수도 있습니다.
- 컨테이너 런타임: `Containerd`, `CRI-O` 등의 CRI에 맞춰 runc 컨테이너를 관리하는 런타임을 사용할 수 있습니다.
  > Docker의 경우 하단에 Containerd가 runc 컨테이너를 관리해 v1.24 이후로 런타임으로서 지원하지 않습니다.

**Addons**
해당 컴포넌트들은 클러스터의 리소스 형태 (DaemonSet, Deployment, etc.)로 배포되어 클러스터 기능을 제공하고 있습니다.
- DNS: 클러스터 DNS는 반드시 필요하며, kubeadm으로 설치할 시 [CoreDNS](https://github.com/coredns/coredns)로 제공됩니다. 클러스터 내부의 컨테이너들은 생성 시 기본 DNS 서버로 해당 DNS 서버를 사용합니다.
- CNI: 클러스터의 네트워크 구성을 책임지며, CNI 규격에 맞게 작동해야합니다. [Cilium](https://cilium.io/use-cases/cni/), [Calico](https://docs.tigera.io/calico/latest/about/), [Flannel](https://github.com/flannel-io/flannel) 등 다양한 프로젝트가 존재하며, Cloud Provider에서 제공하는 종류도 있습니다.

## 프로젝트 구조
> 쿠버네티스는 Go로 개발되어 Go 프로젝트에서 따르는 디렉토리 구조를 많이 따르나, Go 특성상 프로젝트 별로 디렉토리 구조가 상이합니다. 이 부분 감안하고 절대적으로 특정 규칙을 지킬거라 생각하지 않고 접근해야 합니다.

- `api`: 쿠버네티스의 OpenAPI Specification 및 API 문서가 포함되어 있습니다.
- `build`: 프로젝트 빌드 관련 스크립트가 포함되어 있습니다.
- `cluster`: 예전에 사용되던 클러스터 생성 관련 Configuration 파일이 모여있습니다. 더 이상 관리되지 않으니 [공식문서](https://kubernetes.io/docs/setup/)에서 보는 것이 더 정확한 정보일 것으로 보입니다.
- `cmd`: Go 프로젝트에서 흔하게 보이는 디렉토리입니다. 쿠버네티스의 컴포넌트들의 `main` 패키지가 여기서 관리되고 있습니다.
- `pkg`: Go 프로젝트에서 흔하게 보이는 디렉토리입니다. pkg에 보관하면 다른 프로젝트에서 해당 패키지를 import해서 활용할 수 있어, 다른 컴포넌트들이 활용하면서도 외부에 공개해도 문제되지 않는 패키지들이 주로 보관됩니다.
- `plugin`: 
- `staging`:
- `test`:
- `third_party`:
- `vendor`: