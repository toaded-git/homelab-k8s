# homelab-k8s

집 우분투 서버 위에 올린 **쿠버네티스 플랫폼 층**. 순수 학습 목적이고,
최종 목표는 **git push 하면 배포되는 개인 PaaS**("나만의 Railway")다.

같은 기계의 **NAS 층은 별도 저장소 `~/ai-project/homelab` 에서 관리한다.** 이 저장소는 플랫폼만 다룬다.

## 가장 중요한 규칙

**클러스터를 바꾸는 명령은 예외 없이 사용자가 실행한다** (2026-09-18 "무조건 명령어 실행은 다 내가 한다").
Claude 는 설명·개념 정리·공식 문서 안내·작성물 리뷰·결과 해석·힌트·**읽기 전용 확인**·문서 정리를 맡는다.
정답 파일을 먼저 만들어 주지 않는다. 목표와 요구사항을 주고 사용자가 쓴 것을 리뷰한다.

이유: 2026-09-14 Claude 가 k3s 설정을 다 만들어 주자 사용자가 "내가 직접 설치하고 설정해야 배우는거 아니겠음?"
이라고 했다. 플랫폼 층은 배우려고 하는 것이다.

읽기 전용 확인은 Claude 가 해도 된다 (`kubectl get/describe/logs`, 파일 읽기, 버전 조회 등).

## 기기와 접속

| 기기 | 사양 | 역할 |
|---|---|---|
| 우분투 (`uuook-MS-7B89`, 192.168.45.4) | Ryzen 5 3600 (6C/12T), 32GB, GTX 1660 Super, Ubuntu 24.04 | NAS + 플랫폼 |
| 맥미니 | — | 외부 진입점(SSH 포트포워딩), Claude Code 실행 |

- 맥미니 → 우분투: `ssh uuook@192.168.45.4`
- 우분투 → VM: `multipass exec cp1 -- <cmd>` (stdin 을 먹으므로 스크립트에서 `</dev/null` 필요)
- kubectl 은 **cp1 에서만** 된다 (`~/.kube/config`). 워커에는 kubeconfig 가 없어 `localhost:8080 refused` 가 정상
- VM 웹 서비스를 맥에서 보려면 2단 터널:
  `ssh -J uuook@<외부IP>:<포트> uuook@192.168.45.4 -L 30080:10.23.218.234:30080`
  (`-L` 은 **명령을 친 컴퓨터**에 포트를 연다. 맥미니에서 치면 맥미니에 열린다)

## 리소스 현황 (2026-09-29 실측)

호스트 31Gi 중 **14Gi 사용 / 17Gi 여유**, 스왑 8Gi 미사용.

| | 점유 | 비고 |
|---|---|---|
| **k8s VM 3대** (qemu RSS) | **약 11.2 GB** | cp1 4.0 · w1 3.7 · w2 3.5 |
| NAS 컨테이너 7개 합계 | 약 1.3 GB | Immich 가 대부분(약 1.0GB), 나머지는 수백 MB |
| 디스크 SSD | 228G 중 68G (32%) | VM 이미지 20G × 3 포함 |
| 디스크 HDD `/srv/nas` | 1.8T 중 181G (10%) | NAS 전용. 플랫폼은 건드리지 않는다 |

**플랫폼 층이 NAS 보다 9배 가까이 먹는다.** VM 을 최소 사양(2코어·4G·20G)으로 잡아서 그런 것이지
호스트가 부족한 게 아니다. **17Gi 가 남아 있어 확장 여지가 크다.**

확장이 필요해지면 (ArgoCD·Jenkins 처럼 무거운 것을 올릴 때):
- `multipass set local.<vm>.memory=8G` — **VM 을 멈춘 상태에서만** 가능. 버전 지원 여부 확인할 것
- 노드를 한 대 더 추가하는 것도 가능하나, 메모리를 키우는 쪽이 대체로 낫다
- CPU 는 6코어를 VM 3대가 2개씩 나눠 쓰는 중이라 호스트와 겹친다. NAS 트랜스코딩이 돌 때 경합할 수 있다

## 클러스터 현황

**kubeadm 실습 클러스터** — VM 3대, 2026-09-18 구성

| | |
|---|---|
| 노드 | `cp1` 10.23.218.234 (control-plane) · `w1` .70 · `w2` .149 |
| VM | 각 2코어 4GB 20GB, Ubuntu 26.04, multipass NAT `10.23.218.0/24` |
| 버전 | kubeadm·kubelet·kubectl **v1.37.0** (`apt-mark hold`), containerd **2.2.2** |
| CNI | **Calico v3.32.2** (operator), 파드 CIDR `10.244.0.0/16` |
| 스토리지 | `local-path-provisioner` v0.0.37, StorageClass `local-path` (기본 아님, `WaitForFirstConsumer`, `reclaimPolicy: Delete`) |
| 인그레스 | **Traefik** 차트 41.6.0 / 앱 v3.7.13, 네임스페이스 `traefik`, HTTP NodePort **30800** |
| 레지스트리 | `registry:2` NodePort **30500** (emptyDir — 파드 재생성하면 이미지 사라짐) |
| 앱 | `homelab-dashboard` — Ingress `dashboard.home`, NodePort 30080 |

### 주소 대역 (겹치면 안 됨)

```
192.168.45.0/24    집 LAN
10.23.218.0/24     multipass NAT (VM)
10.244.0.0/16      파드
10.96.0.0/12       서비스 ClusterIP
172.17~21          Docker 브리지 (NAS 층)
```

**Calico 기본 IP 풀이 `192.168.0.0/16` 이라 집 LAN 과 겹친다.** 설치 때 `custom-resources.yaml` 에서
직접 고쳤다. 문서는 자동 감지된다고 하지만 실제로는 거부당한다.

### 우분투 본체에 kubeadm 을 올리지 않는 이유

Docker 용 containerd 가 `disabled_plugins = ["cri"]` 라 CRI 를 켜고 재시작해야 하는데,
그러면 **NAS 컨테이너가 같이 재시작된다.** 그래서 VM 으로 분리했다.

## 저장소 규칙

- 플랫폼 매니페스트는 이 저장소 `platform/` 이 원본이다. **작업본은 cp1 `/home/ubuntu/`** 이므로,
  거기서 고쳤으면 여기로 다시 가져온다: `multipass exec cp1 -- cat /home/ubuntu/<f>.yaml > platform/<...>/<f>.yaml`
- 비밀값(개인키가 든 Secret YAML 등)은 커밋하지 않는다. `.gitignore` 에 `platform/**/*secret*.yaml`
- 외부 매니페스트·차트는 **버전을 박아서** 쓴다 (`--version 41.6.0`, URL 에 태그). 재현 가능해야 한다
- 할 일은 `docs/TODO.md`, 배운 것은 `docs/til/` 에 일자별로 남긴다. 플랫폼 작업을 한 날은 해당 일자 파일을 추가한다
- **저장소 → 클러스터 배포는 아직 수동이다** (cp1 에서 고치고 여기로 되가져옴). 자동화 스크립트를 따로 만들지 않고,
  2026-09 넷째 주에 올릴 **ArgoCD 로 한 번에 해결하기로 했다** (사용자 결정 2026-09-29)

## NAS 층과의 경계

NAS 층(Jellyfin·Samba·Immich·restic·Uptime Kuma)은 같은 기계에서 Docker Compose 로 돈다.
**두 층은 서로 의존하지 않는다.** 플랫폼은 부수고 다시 세워도 되고, NAS 는 안정적으로 유지한다.

- 플랫폼은 `/srv/nas` 를 마운트하지 않는다. k3s 를 올릴 때도 `/var/lib/rancher`(SSD)만 쓴다
- 영화 라이브러리(`/srv/nas/media`)는 외부 공개·타인 공유 금지. **플랫폼 층이나 공개 서비스와 연결하지 않는다**
- 우분투 `/etc/docker/daemon.json` 은 NAS 층 것이다. 클러스터 레지스트리 때문에 `insecure-registries` 를 넣었는데,
  **nvidia 런타임 블록을 지우면 Jellyfin·Immich GPU 가 죽는다.** 덮어쓰지 말고 합칠 것
- 파드에서 집 LAN(192.168.45.4:22 등)으로 나가는 것은 현재 열려 있다. 공개 서비스를 올리게 되면
  **NetworkPolicy 로 막아야 한다** (Calico 를 고른 이유)

## 방향

- 관심사는 AI 보다 인프라. LLM 서버는 하지 않기로 했다
- 회사에서 k8s 를 쓰지 않아 홈랩으로 배우는 중. 주력은 백엔드(Java)이므로 개념 설명은
  배포·리버스 프록시·환경변수·CI/CD·로드밸런서 경험에 빗대면 빠르다. 웹 개발 기초는 설명하지 않아도 된다
- 경로: kubeadm 실습(진행 중) → 우분투 본체에 k3s → git push 배포 자동화
- 저장소는 나중에 공개할 수 있다. 공개 전에 외부 SSH 포트 같은 민감 정보를 훑을 것
  (집 LAN 사설 IP 자체는 무해하고, "Calico 기본값이 집 LAN 과 겹쳤다" 같은 서술에는 필요하다)

## 함정 모음

설치보다 **기본값이 내 환경과 어긋나는 지점**에서 시간이 갔다. 자세한 것은 `docs/til/`.

- **틀린 설정이 에러를 안 낸다.** containerd 1.x 경로(`io.containerd.grpc.v1.cri`)로 고치면 무시되고,
  Helm 값도 없는 키(`service.type` vs `service.spec.type`)면 조용히 넘어간다.
  → `--dry-run=server`, `helm show values`, `describe` 의 `Mounts:` 로 **반영됐는지 확인**하는 절차를 늘 끼울 것
- multipass 데몬이 물리면 `sudo snap restart multipass` (qemu 가 데몬 자식이라 VM 도 같이 꺼짐)
- VM 스냅샷은 **정지 상태에서만** 찍힌다. `kubeadm reset` 보다 스냅샷 복원이 깨끗하다
- VM 재생성·호스트 재부팅 시 IP 가 바뀔 수 있다 (2026-09-27 재부팅에서는 그대로였고 클러스터가 자가 복구됨.
  파드 IP 는 바뀌었지만 Service 와 iptables 규칙이 따라갔다)
