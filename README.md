# homelab-k8s

집 서버 한 대 위에 손으로 세운 쿠버네티스 클러스터와, 거기에 올린 앱들.

회사에서 쿠버네티스를 쓰지 않아 집에서 배우는 중이다.
**설치·설정·명령은 전부 직접 실행했고**, 막힌 곳과 그 이유를 날마다 [`docs/til/`](docs/til/)에 남긴다.
최종 목표는 **git push 하면 배포되는 개인 PaaS**.

## 구성

| | |
|---|---|
| 호스트 | Ubuntu 24.04, Ryzen 5 3600, 32GB, GTX 1660 Super |
| 노드 | multipass VM 3대 (`cp1`·`w1`·`w2`), 각 2코어 4GB 20GB, Ubuntu 26.04 |
| 쿠버네티스 | kubeadm · kubelet · kubectl **v1.37.0** |
| 런타임 | containerd **2.2.2** (`SystemdCgroup = true`) |
| CNI | **Calico v3.32.2**, 파드 CIDR `10.244.0.0/16` |
| 인그레스 | **Traefik** v3.7.13 (Helm 차트 41.6.0) |
| 스토리지 | **local-path-provisioner** v0.0.37 |
| 레지스트리 | `registry:2` (클러스터 안, NodePort 30500) |

같은 기계에서 NAS(Jellyfin·Samba·Immich·restic·Uptime Kuma)가 Docker Compose 로 돈다.
**두 층은 서로 의존하지 않는다.** 클러스터를 부수고 다시 세워도 NAS 는 영향을 받지 않는다.

## 지금 돌아가는 것

```
맥에서 git push ──> 우분투 bare 저장소 ──> clone ──> docker build (23.5MB)
                                                         │
                                                 docker push (평문 HTTP 예외)
                                                         ▼
                       ┌─── VM 3대 · 10.23.218.0/24 ──────────────────┐
                       │                                             │
                       │   registry:2        NodePort 30500          │
                       │        ▲ pull (containerd certs.d)          │
                       │   Traefik           NodePort 30800          │
                       │        │ Host 헤더로 라우팅                  │
                       │   dashboard 파드    dashboard.home           │
                       │     /config  ← ConfigMap                    │
                       │     /secrets ← Secret (0400, tmpfs)         │
                       │     /data    ← PVC (local-path, RWO)        │
                       │     readiness · liveness → /api/health      │
                       └─────────────────────────────────────────────┘
```

**이미지 안에는 바이너리뿐이다.** 설정도 비밀값도 저장소도 전부 바깥에서 주입된다.

## 저장소 구조

```
platform/
├── registry/       클러스터 안 이미지 레지스트리
├── traefik/        인그레스 컨트롤러 (Helm values)
├── storage/        local-path-provisioner
└── dashboard/      직접 만든 앱 — Deployment · Service · Ingress · PVC
docs/
├── til/            일별 기록: 한 일 / 막힌 곳 / 배운 것
└── TODO.md
```

매니페스트는 버전을 박아서 쓴다. 외부 차트·매니페스트도 태그를 고정한다 — 재현 가능해야 하므로.

## 기록

[`docs/til/`](docs/til/) 에 날마다 남긴다. 설치 절차보다 **기본값이 내 환경과 어긋난 지점**이 대부분이다.

| | | |
|---|---|---|
| Day 1 | [설치하기 전에 멈춰 선 날](docs/til/day1-2026-09-14-plan.md) | 호스트에 직접 못 올리는 이유 |
| Day 2 | [커널부터 맞추기](docs/til/day2-2026-09-17-node-prereq.md) | `br_netfilter`·`ip_forward` 가 왜 필요한가 |
| Day 3 | [클러스터가 서다](docs/til/day3-2026-09-18-cluster-up.md) | **Calico 기본 IP 풀이 집 LAN 과 충돌** |
| Day 4 | [Service 를 믿지 말고 확인하기](docs/til/day4-2026-09-28-service-and-image.md) | Service 는 확률 분기 iptables 규칙 |
| Day 5 | [사설 레지스트리와 첫 배포](docs/til/day5-2026-09-29-registry-and-deploy.md) | containerd 2.x `certs.d`, CrashLoop 진단, 인그레스 |

몇 가지 예:

- Calico 문서는 "kubeadm 이면 파드 CIDR 을 자동 감지하므로 수정 불필요"라고 하지만 실제로는 거부당한다.
  기본 IP 풀 `192.168.0.0/16` 이 가정용 공유기 대역과 정면으로 겹친다
- `kubernetes/ingress-nginx` 는 2026-03 에 아카이브됐다. 그래서 Traefik 을 골랐다
- containerd 2.x 는 레지스트리 설정 경로가 1.x 와 달라서, 검색되는 예제를 그대로 쓰면 **오류 없이 무시된다**
- Helm 도 없는 키를 설정하면 조용히 넘어간다 (`service.type` vs `service.spec.type`)

공통점은 **틀린 설정이 에러를 내지 않는다**는 것이다. 그래서 `--dry-run=server`,
`helm show values`, `kubectl describe` 의 `Mounts:` 로 반영 여부를 확인하는 절차를 늘 끼운다.

## 앞으로

- [ ] ArgoCD — GitOps. 지금은 cp1 에서 고치고 저장소로 되가져오는 수동 흐름이다
- [ ] Calico NetworkPolicy — CNI 를 Calico 로 고른 원래 이유
- [ ] 우분투 본체에 k3s — kubeadm 으로 손수 한 것을 k3s 가 얼마나 줄여 주는지 비교
- [ ] git push → 빌드 → 배포 자동화
