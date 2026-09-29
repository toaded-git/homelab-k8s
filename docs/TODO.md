# TODO — 플랫폼 층

끝난 항목은 체크하고, 확정된 상태는 `CLAUDE.md` 에 반영한다.
배운 것·막힌 것은 `docs/til/` 에 일자별로 남긴다.

**클러스터를 바꾸는 명령은 사용자가 실행한다.** Claude 는 읽기 전용 확인만 한다.

- [ ] kubeadm 클러스터 (VM, 직접) — 2026-09-16 결정: "큰 것부터", k8s 운영 층을 먼저 배우고 그다음 k3s
  - VM: multipass (우분투 KVM 위), `cp1`·`w1`·`w2` 각 2코어·4G·20G, Ubuntu 26.04, NAT 대역 10.23.218.0/24
  - [x] 사전 준비 (2026-09-17): `overlay`·`br_netfilter` 모듈 + `/etc/modules-load.d/k8s.conf`, sysctl 3개 + `/etc/sysctl.d/k8s.conf`, swap 없음, 재부팅 후 유지 확인
  - [x] containerd (2026-09-18): 우분투 apt `containerd` 2.2.2, `/etc/containerd/config.toml` 은 `containerd config default` 로 생성(패키지가 설정 파일을 안 넣음), `SystemdCgroup = true`, CRI 플러그인 ok
  - [x] kubeadm·kubelet·kubectl v1.37.0 설치 + `apt-mark hold` (2026-09-18). 스냅샷 `prereq`(모듈·sysctl) → `ready`(도구까지) 3대 모두
  - 스냅샷은 VM 을 멈춘 상태에서만 찍힘. `kubeadm init` 실패 시 reset 보다 `ready` 복원이 깨끗함
  - [ ] `kubeadm init` → CNI → 워커 join
    - CNI 결정 (2026-09-18): **Calico** (NetworkPolicy 실습 목적). 파드 CIDR 은 `10.244.0.0/16`
    - Calico v3.32.2, operator 방식: `v1_crd_projectcalico_org.yaml` → `tigera-operator.yaml` → `custom-resources.yaml` (raw.githubusercontent.com/projectcalico/calico/v3.32.2/manifests/)
    - **`custom-resources.yaml` 의 기본 IP 풀이 192.168.0.0/16 이라 집 LAN(192.168.45.0/24)과 겹침.** 문서는 "kubeadm 이면 Calico 가 실제 파드 CIDR 을 자동 감지하므로 수정 불필요"라고 하지만, 설치 후 `kubectl get ippools -o wide` 로 10.244.0.0/16 인지 반드시 확인할 것 (아니면 파일에서 직접 수정)
    - cp1 API 주소: 10.23.218.234 (multipass DHCP 라 재생성 시 바뀜)
    - [x] 3노드 구성 완료 (2026-09-18): Calico v3.32.2, 파드 CIDR 10.244.0.0/16 확인, nginx Deployment·ReplicaSet·Pod·Service(ClusterIP/NodePort) 실습
    - 2026-09-27 우분투 본체 재부팅에서 클러스터가 스스로 복구됨: multipass VM 자동 시작 → containerd → kubelet → 파드 재생성. **VM IP 는 그대로였지만 보장은 아님**. 파드 IP 는 바뀌었고 Service·iptables 규칙이 자동으로 따라감
  - 함정: multipass 데몬이 restart·force stop·delete/launch 에서 반복적으로 물림 (VM 은 정상인데 상태가 Starting 에서 안 넘어옴). `sudo snap restart multipass` 로 해소. 재발하면 libvirt 전환 고려
  - 함정: VM 재생성·호스트 재부팅 시 IP 바뀜 → kubeadm 클러스터가 깨질 수 있음
- [ ] k3s 설치 (직접, kubeadm 실습 뒤) — 학습 목적. 설정 파일·절차는 사용자가 작성해 `platform/k3s/` 에 둔다
  - 사용자 결정 (2026-09-14): 내장 containerd, Traefik 끄고 인그레스 직접 설치, ufw 유지 + 규칙, kubectl 은 우분투에서
  - 사전 점검: 포트 80·443·6443·10250·8472 비어 있음, Docker 대역과 10.42/10.43 안 겹침, LAN IP 수동 고정, swap 은 k3s 기본 허용
  - [ ] 요구사항·ufw 문서 읽기 → 설정 파일 작성 → 설치 → kubeconfig
  - [ ] 확인: 노드 Ready, 시스템 파드, 파드 DNS·인터넷 (ufw FORWARD 영향), runtimeClass nvidia, NAS 서비스 영향 없음
- [x] 인그레스 컨트롤러 (2026-09-29) — **Traefik** 차트 41.6.0 / 앱 v3.7.13, Helm v4.3.0, 네임스페이스 `traefik`
  - `kubernetes/ingress-nginx` 는 2026-03 아카이브(마지막 릴리스 2026-03-19). 살아 있는 것 중 Ingress + Gateway API 를 둘 다 지원하는 Traefik 선택
  - 진입: HTTP NodePort **30800**, HTTPS **30843**. IngressClass `traefik` 가 기본값
  - 대시보드 Ingress: `dashboard.home` → Service `dashboard:8080`. `curl -H "Host: dashboard.home"` 정상, 다른 호스트는 404 확인
  - [ ] Traefik Service 가 아직 LoadBalancer 라 EXTERNAL-IP `<pending>`. 차트 41.x 키는 `service.type` 이 아니라 **`service.spec.type`** — 값 파일 고쳐서 `helm upgrade`
  - [ ] Gateway API(HTTPRoute) 로도 같은 라우팅 해보기 (같은 컨트롤러로 비교 가능)
  - [ ] TLS: Secret 에 인증서 넣고 `spec.tls`, 나중에 cert-manager
- [ ] 앱 배포 파이프라인 (진행 중) — 대상 앱 `homelab-dashboard` (Go+React, 단일 컨테이너, 8080)
  - 매니페스트는 저장소 `platform/registry/`·`platform/dashboard/` 에 있다. cp1 `/home/ubuntu/` 사본이 아직 작업본이라 고치면 저장소로 다시 가져올 것
  - [x] git 원격: 우분투 `~/git/homelab-dashboard.git` (bare). 맥에서 push, 커밋 `f816376`. 맥/우분투 소스는 동일했음
  - [x] 우분투에서 `~/build/homelab-dashboard` 로 clone 후 `docker build -t homelab-dashboard:v1` → 23.5MB
  - [x] 클러스터 안 레지스트리: `registry:2`, NodePort **30500**, emptyDir (파드 재생성하면 이미지 날아감 → 나중에 PVC)
  - [x] 우분투 `/etc/docker/daemon.json` 에 `insecure-registries` + `live-restore` 추가, nvidia 런타임 유지 확인 (2026-09-29). 백업 `daemon.json.bak`
  - [x] `docker push 10.23.218.234:30500/homelab-dashboard:v1` → `_catalog` 확인
  - [x] 노드 3대 containerd `certs.d` (2026-09-29): `config.toml` 54행 `config_path = '/etc/containerd/certs.d'` + `certs.d/10.23.218.234:30500/hosts.toml`. containerd 재시작해도 실행 중 파드는 안 죽음
  - [x] 대시보드 Deployment(레플리카 2, w1·w2 분산) + Service(NodePort 30080)
  - [x] ConfigMap `dashboard-config` — `config.yaml`·`known_hosts` 를 `/config` 에 마운트. 앱 기동 성공(`listening addr=:8080 hosts=3`)
  - [x] Secret `dashboard-ssh` — `id_ed25519` 를 `/secrets` 에 마운트 (`defaultMode: 0400`). **이 YAML 은 커밋 금지**
    - `config.yaml` 의 `key_path` 를 `/secrets/id_ed25519` 로 바꿔둠 (ConfigMap 과 Secret 을 같은 경로에 마운트할 수 없어서)
  - [x] 파드 → 우분투 호스트(192.168.45.4:22)·맥미니 SSH 됨. ufw 영향 없었음
  - [x] 스토리지: `local-path-provisioner` v0.0.37 설치 → StorageClass `local-path` (기본 아님, `WaitForFirstConsumer`, `reclaimPolicy: Delete`)
  - [x] PVC `dashboard-data` 1Gi RWO → `/data`. **레플리카 1 로 줄임** (RWO + 로컬 디스크)
  - [x] 2026-09-29 동작 확인: `http://10.23.218.234:30080` HTTP 200, `/api/health` ok, 로그에 ERROR 없음
  - [x] readiness/liveness probe (`/api/health`) + 리소스 requests/limits → QoS `Burstable` (2026-09-29)
    - readiness 는 빡빡하게(3s/5s), liveness 는 느슨하게(20s/20s/3회). liveness 를 조이면 기동 중인 파드를 죽여 무한 재시작
  - [ ] 레플리카 2 + RWO 가 실제로 어떻게 깨지는지 실험 (Pending 인지, 같은 노드로 몰리는지)
  - [ ] 레지스트리 저장소를 emptyDir → PVC 로
  - 우분투 `~/homelab-dashboard` 는 7월에 복사해둔 예전 폴더. `config/config.yaml`·`id_ed25519`·`known_hosts`·`data/`(4.8MB) 가 재료
- [ ] "나만의 Railway" 설계: git 서버, 이미지 레지스트리, 빌드, 배포 흐름
