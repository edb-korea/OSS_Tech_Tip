# Red Hat 계열에서 k3s 설치하기

RHEL 8/9 및 호환 배포판(Rocky Linux, AlmaLinux)에 **k3s**로 Kubernetes 환경을 구성하는 절차입니다. ClickHouse(Altinity Kubernetes Operator, Altinity.Cloud Anywhere) 실행 환경으로 쓰는 것을 염두에 두었습니다.

> **참고**: 이 문서는 일반적인 k3s 설치 절차를 정리한 것이며, 작성 시점에 실제 서버에서 검증하지는 않았습니다. 설치 전 [k3s 공식 문서](https://docs.k3s.io/)의 요구사항과 옵션을 확인하세요.

---

## 1. 사전 요구사항

| 항목 | 권장 사항 |
|---|---|
| OS | RHEL 8/9, Rocky Linux 8/9, AlmaLinux 8/9 |
| 권한 | `root` 또는 `sudo` 가능 계정 |
| CPU / 메모리 | PoC 최소 2 vCPU / 2 GB. ClickHouse를 올리려면 4 vCPU / 8 GB 이상 권장 |
| 디스크 | ClickHouse 데이터용으로 충분한 용량 (로컬 NVMe/SSD 권장) |
| 네트워크 | 설치 시 인터넷 접속 필요 (폐쇄망은 [부록 A](#부록-a-폐쇄망air-gapped-설치) 참고) |
| 호스트명 | 노드마다 고유한 호스트명 |

서버 정보를 먼저 확인합니다.

```bash
cat /etc/redhat-release
uname -r
getenforce
```

---

## 2. 사전 설정

### 2-1. 방화벽(firewalld)

방화벽 처리는 두 가지 중 하나를 선택합니다.

**방법 A. firewalld 비활성화 (PoC에서 가장 간단)**

```bash
sudo systemctl disable --now firewalld
```

**방법 B. firewalld 유지 (운영 권장)**

Pod/Service 대역을 trusted 존에 등록하고 API 서버 포트를 엽니다.

```bash
sudo firewall-cmd --permanent --zone=trusted --add-source=10.42.0.0/16   # Pod CIDR (k3s 기본값)
sudo firewall-cmd --permanent --zone=trusted --add-source=10.43.0.0/16   # Service CIDR (k3s 기본값)
sudo firewall-cmd --permanent --add-port=6443/tcp                        # Kubernetes API
sudo firewall-cmd --permanent --add-port=10250/tcp                       # kubelet
sudo firewall-cmd --reload
```

멀티 노드로 확장할 경우 아래 포트도 필요합니다.

| 포트 | 용도 |
|---|---|
| 8472/udp | Flannel VXLAN (노드 간 Pod 통신) |
| 2379-2380/tcp | 내장 etcd (HA 서버 구성 시) |

### 2-2. nm-cloud-setup (클라우드 VM인 경우)

RHEL 8.4 이상의 클라우드 이미지에서 `nm-cloud-setup`이 켜져 있으면 k3s 네트워크와 충돌할 수 있습니다. 활성화되어 있다면 끄고 재부팅합니다.

```bash
sudo systemctl disable --now nm-cloud-setup.service nm-cloud-setup.timer
sudo reboot
```

### 2-3. SELinux

SELinux는 **enforcing 상태로 두어도 됩니다.** 설치 스크립트가 `k3s-selinux` 정책 패키지를 함께 설치합니다. (폐쇄망에서는 RPM을 직접 설치해야 합니다.)

### 2-4. (선택) 시간 동기화

```bash
sudo systemctl enable --now chronyd
chronyc tracking
```

---

## 3. k3s 설치 (단일 노드)

```bash
curl -sfL https://get.k3s.io | sudo sh -
```

설치 스크립트를 먼저 확인하고 싶다면 파일로 받아 내용을 본 뒤 실행합니다.

```bash
curl -sfL https://get.k3s.io -o /tmp/k3s-install.sh
less /tmp/k3s-install.sh
sudo sh /tmp/k3s-install.sh
```

### 버전 고정 설치

재현 가능한 환경을 위해 운영에서는 버전을 고정하는 것을 권장합니다. 설치 가능한 버전은 [k3s 릴리스 페이지](https://github.com/k3s-io/k3s/releases)에서 확인합니다.

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION="<k3s-version>" sudo sh -
```

### 자주 쓰는 설치 옵션

`INSTALL_K3S_EXEC`로 서버 옵션을 전달합니다.

```bash
# sudo 없이 kubectl을 쓰기 위해 kubeconfig 권한 완화
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--write-kubeconfig-mode 644" sudo sh -

# Traefik(기본 Ingress) 비활성화 - 다른 Ingress를 쓸 때
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik" sudo sh -
```

---

## 4. 설치 확인

```bash
sudo systemctl status k3s
sudo /usr/local/bin/k3s kubectl get nodes
sudo /usr/local/bin/k3s kubectl get pods -A
```

노드 상태가 `Ready`이고 `kube-system`의 Pod가 `Running` 또는 `Completed`이면 정상입니다.

> RHEL의 `sudo`는 `secure_path`에 `/usr/local/bin`이 없을 수 있어 `sudo k3s ...`가 "command not found"로 실패할 수 있습니다. 이 경우 위처럼 전체 경로(`/usr/local/bin/k3s`)를 사용하세요.

---

## 5. kubectl 설정

k3s는 `/usr/local/bin/kubectl`(k3s 래퍼)을 설치합니다. 일반 사용자로 쓰려면 kubeconfig를 복사합니다.

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown "$(id -u):$(id -g)" ~/.kube/config
chmod 600 ~/.kube/config

kubectl get nodes
```

원격 PC에서 접속하려면 복사한 파일의 `server: https://127.0.0.1:6443`을 서버 IP 또는 호스트명으로 바꿉니다.

### (선택) Helm 설치

```bash
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

---

## 6. 스토리지 확인

k3s는 `local-path` StorageClass를 기본으로 제공합니다.

```bash
kubectl get storageclass
```

- 기본값은 노드의 `/var/lib/rancher/k3s/storage` 아래에 데이터를 저장합니다.
- ClickHouse처럼 I/O가 큰 워크로드는 **전용 디스크를 별도 경로에 마운트**하고, local-path 경로를 그쪽으로 지정하거나 별도 CSI 드라이버를 쓰는 것을 검토하세요.
- 노드 장애 시 로컬 볼륨 데이터는 다른 노드로 옮겨지지 않습니다. 운영에서는 복제(ClickHouse replica)와 백업 설계가 필요합니다.

---

## 7. 노드 추가 (멀티 노드)

### 7-1. 서버(첫 노드)에서 토큰 확인

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

### 7-2. 워커 노드에서 에이전트 설치

각 워커 노드에 2장의 사전 설정(방화벽 등)을 적용한 뒤 실행합니다.

```bash
curl -sfL https://get.k3s.io | K3S_URL="https://<server-ip>:6443" K3S_TOKEN="<node-token>" sudo sh -
```

### 7-3. 서버에서 확인

```bash
kubectl get nodes -o wide
```

> 컨트롤 플레인 HA(서버 3대, 내장 etcd)는 `--cluster-init` 옵션 등 별도 구성이 필요합니다. [k3s HA 문서](https://docs.k3s.io/datastore/ha-embedded)를 참고하세요.

---

## 8. 다음 단계: ClickHouse 환경

### Altinity Kubernetes Operator 설치

설치 번들 주소와 버전은 [Altinity 공식 문서/저장소](https://github.com/Altinity/clickhouse-operator)에서 확인하세요.

```bash
kubectl apply -f https://raw.githubusercontent.com/Altinity/clickhouse-operator/master/deploy/operator/clickhouse-operator-install-bundle.yaml
kubectl get pods -n kube-system | grep clickhouse-operator
```

### Altinity.Cloud Anywhere(BYOK) 연결

ACM에서 환경을 만들고 발급된 일회용 토큰으로 `altinitycloud-connect`를 등록합니다. 이 클러스터에서 Altinity 관리 플레인으로 **outbound 연결**이 가능해야 합니다. 자세한 절차는 [Connecting to Altinity.Cloud Anywhere](https://docs.altinity.com/altinitycloudanywhere/bring-your-own-kubernetes-byok/connecting)를 참고하세요.

---

## 9. 문제 해결

| 증상 | 확인 사항 |
|---|---|
| 노드가 `NotReady` | `sudo journalctl -u k3s -f`로 로그 확인. 방화벽, `nm-cloud-setup` 충돌 여부 점검 |
| Pod 간 통신 불가 | firewalld trusted 존에 `10.42.0.0/16`, `10.43.0.0/16` 등록 여부 확인 |
| 설치 중 SELinux 관련 오류 | `getenforce`, `sudo ausearch -m avc -ts recent`로 거부 로그 확인. 인터넷 차단 시 `k3s-selinux` RPM 수동 설치 |
| `sudo k3s` 명령을 찾지 못함 | `/usr/local/bin/k3s` 전체 경로 사용 |
| 이미지 Pull 실패 | 프록시/레지스트리 접근 확인. 사내 레지스트리는 `/etc/rancher/k3s/registries.yaml`로 설정 |
| 서비스 로그 확인 | 서버: `journalctl -u k3s`, 에이전트: `journalctl -u k3s-agent` |

---

## 10. 삭제

```bash
# 서버 노드
sudo /usr/local/bin/k3s-uninstall.sh

# 에이전트(워커) 노드
sudo /usr/local/bin/k3s-agent-uninstall.sh
```

> 삭제하면 클러스터 데이터와 로컬 스토리지의 PV 데이터도 함께 제거될 수 있습니다. 필요한 데이터는 먼저 백업하세요.

---

## 부록 A. 폐쇄망(air-gapped) 설치

인터넷이 차단된 환경에서는 아래 파일을 인터넷 가능한 PC에서 받아 서버로 옮겨야 합니다. 구체적인 파일명과 절차는 [k3s Air-Gap 문서](https://docs.k3s.io/installation/airgap)를 따르세요.

1. k3s 바이너리
2. air-gap 이미지 아카이브 (`k3s-airgap-images-<arch>.tar.zst`)
3. 설치 스크립트 (`install.sh`)
4. `k3s-selinux` 및 의존 RPM (SELinux enforcing 사용 시)

아카이브를 `/var/lib/rancher/k3s/agent/images/`에 둔 뒤 `INSTALL_K3S_SKIP_DOWNLOAD=true`로 설치 스크립트를 실행합니다.

## 부록 B. 선택 가이드

| 상황 | 권장 |
|---|---|
| PoC, 개발, 단일 노드 | **k3s** (본 문서) |
| Red Hat 공식 지원이 필요한 엣지/단일 노드 | MicroShift |
| 대규모/정식 멀티 노드 | kubeadm 또는 OpenShift |
