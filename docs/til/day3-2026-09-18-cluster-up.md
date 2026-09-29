# Day 3 (2026-09-18) — 클러스터가 서다

← [Day 2](day2-2026-09-17-node-prereq.md) · [목차](README.md) · [다음: Day 4](day4-2026-09-28-service-and-image.md)

## 한 일

- containerd 2.2.2 설치, `SystemdCgroup = true`
- kubeadm·kubelet·kubectl v1.37.0 설치 후 `apt-mark hold`
- VM 스냅샷 `prereq`(모듈·sysctl) → `ready`(도구까지) 3대 모두
- `kubeadm init --pod-network-cidr=10.244.0.0/16`
- Calico v3.32.2 operator 설치 (CRD → `tigera-operator.yaml` → `custom-resources.yaml`)
- 워커 2대 join
- nginx 로 Deployment → ReplicaSet → Pod, Service(ClusterIP / NodePort) 실습

## 막힌 곳

### apt 로 깐 containerd 에 `config.toml` 이 아예 없었다

패키지가 설정 파일을 넣지 않는다. 직접 생성해야 한다.

```sh
containerd config default | sudo tee /etc/containerd/config.toml
```

### 워커 join 실패

```
[ERROR IsPrivilegedUser]
```

`sudo` 를 빼먹었다. 에러 이름이 원인을 그대로 말해주는 좋은 예.

### Calico 가 설치를 거부했다

```
IPPool 192.168.0.0/16 is not within the platform's configured pod network CIDR(s) [10.244.0.0/16]
```

Calico 공식 `custom-resources.yaml` 의 기본 IP 풀이 `192.168.0.0/16` 인데, **우리 집 LAN(192.168.45.0/24)과 겹친다.**
그래서 파드 CIDR 을 `10.244.0.0/16` 으로 잡고 `kubeadm init` 을 했다.

문서에는 "kubeadm 환경이면 Calico 가 실제 파드 CIDR 을 자동 감지하므로 수정 불필요"라고 적혀 있지만,
**실제로는 거부당했다.** 파일에서 직접 고쳐서 해결.

```sh
sed -i 's|192.168.0.0/16|10.244.0.0/16|' custom-resources.yaml
kubectl apply -f custom-resources.yaml
kubectl get ippools -o wide   # 확인
```

## 배운 것

### cgroup 드라이버는 kubelet 과 런타임이 같아야 한다

systemd 로 부팅하는 배포판에서는 둘 다 systemd 여야 한다.
`SystemdCgroup = true` 를 빠뜨리면 자원 제한이 엉뚱하게 동작하거나 노드가 불안정해진다.
두 프로세스가 같은 cgroup 트리를 서로 다른 방식으로 만지면 안 되기 때문.

### `apt-mark hold`

클러스터 구성 요소가 `apt upgrade` 에 딸려 올라가면 안 된다.
쿠버네티스 버전 업은 순서와 절차가 있는 작업이라 **의도적으로만** 해야 한다.

### 컨트롤 플레인도 파드다

apiserver·etcd·controller-manager·scheduler 는 `/etc/kubernetes/manifests/` 에 YAML 로 놓인 **static pod** 고,
kubelet 이 그 디렉터리를 직접 읽어서 띄운다.

그래서 **apiserver 가 없어도 뜬다** (닭-달걀 문제를 이렇게 푼다).
`kubectl delete` 로 지워도 kubelet 이 다시 만든다. 지우려면 파일을 옮겨야 한다.

### 네트워크 대역은 설치 전에 정해야 한다

파드 CIDR 이 집 LAN 과 겹치면 나중에 고치기가 매우 어렵다.
문서·예제가 `192.168.0.0/16` 을 기본값으로 쓰는 경우가 많은데, **가정용 공유기 대역과 정면으로 충돌한다.**
클라우드에서 실습하면 안 만나고 집에서 하면 바로 만나는 종류의 함정.

### VM 스냅샷은 정지 상태에서만 찍힌다

`kubeadm reset` 으로 되돌리는 것보다 `ready` 스냅샷 복원이 훨씬 깨끗하다.
reset 은 남기는 게 있어서 "지운 줄 알았는데 남아 있는" 상태를 만든다.

---
← [Day 2](day2-2026-09-17-node-prereq.md) · [목차](README.md) · [다음: Day 4](day4-2026-09-28-service-and-image.md)
