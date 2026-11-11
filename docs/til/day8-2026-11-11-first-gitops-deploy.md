# Day 8 (2026-11-11) — git push 로 첫 배포

← [Day 7](day7-2026-11-02-argocd-login.md) · [목차](README.md)

## 한 일

- 맥에서 매니페스트 한 줄 수정 → push → ArgoCD 감지 → sync. **cp1 에서 kubectl 없이** 클러스터를 바꿨다
- MetalLB 를 시작하려다 방향 점검 — 결정은 주말로

## 흐름이 뒤집혔다

```
전: cp1 에서 YAML 수정 → kubectl apply → 맥 저장소로 되가져옴
후: 맥 저장소에서 수정 → git push origin → ArgoCD 감지 → sync
```

`platform/dashboard/` 는 이제 **git 이 원본**이다. cp1 의 사본은 더 이상 쓰지 않는다.

## 무엇을 바꿨나 — 일부러 무해한 것

흐름이 한 바퀴 도는지만 보면 되므로, 앱 동작에 영향이 없는 **라벨 한 줄**을 넣었다.
`managed-by: argocd` 는 아무 기능이 없는 이름표다. ArgoCD 도 쿠버네티스도 읽지 않는다.
(비슷한 이름의 관례 `app.kubernetes.io/managed-by` 도 정보용일 뿐이다. Helm 은 여기에 `Helm` 을 적는다.)

### Deployment 안의 `labels` 는 세 군데다

| 위치 | 바꾸면 |
|---|---|
| `metadata.labels` (Deployment 자신) | 파드 재시작 없음 ← 여기에 넣음 |
| `spec.template.metadata.labels` (파드 템플릿) | 템플릿이 바뀌어 **롤아웃** |
| `spec.selector.matchLabels` | **불변 필드** — sync 실패 |

## ArgoCD 는 라벨이 아니라 git 과의 차이로 감지한다

```
3분마다(또는 REFRESH) origin 을 당겨옴 → platform/dashboard/ 렌더링 → 클러스터와 리소스별 비교 → 다르면 OutOfSync
```

push 직후 상태:

```
Deployment/dashboard:            OutOfSync   ← 바꾼 것 하나만
Service/dashboard:               Synced
Ingress/dashboard:               Synced
PersistentVolumeClaim/dashboard-data: Synced
```

앱 전체가 아니라 **리소스 단위**로 비교한다. revision 도 방금 커밋(`b44f88f`)으로 바뀌어 있었다.

## sync 결과 — 예측대로 재시작 없음

| | |
|---|---|
| Deployment 라벨 | `{"app":"dashboard","managed-by":"argocd"}` |
| 파드 | AGE 42d 그대로 |
| ReplicaSet | `665d87d486` 그대로 → 롤아웃 없음 |

health 는 여전히 Progressing — Ingress ADDRESS 가 비어서다 ([Day 7](day7-2026-11-02-argocd-login.md)).

## 이제 cp1 에서 `kubectl apply` 하면 안 되는 이유

push 전에 같은 파일로 `kubectl diff` 를 해 보니 이런 줄이 섞여 있었다.

```diff
-    argocd.argoproj.io/tracking-id: dashboard:apps/Deployment:default/dashboard
+    managed-by: argocd
```

git 파일에는 ArgoCD 의 소유 표시가 없으니, **손으로 apply 하면 그 표시가 지워진다.** ArgoCD sync 는 자기가 다시 붙인다.
손으로 바꾼 내용도 결국 git 과 달라져 OutOfSync 가 된다. 바꾸고 싶으면 git 에서.

## 자잘한 것

- vi 로 열어 둔 사이 저장소에 `.dashboard-deploy.yaml.swp` 가 생겼다. `git add .` 대신 파일을 지정해서 add 했다.
  `.gitignore` 에 `*.swp` 추가

## 방향 점검: MetalLB 를 하려다 멈춤

### 대역이 없다

```
multipass DHCP: 10.23.218.2 ~ 10.23.218.254 (전체), 임대 무기한
```

MetalLB L2 는 노드와 같은 서브넷 IP 를 나눠 줘야 하는데 비는 대역이 없다. 끝쪽(.250~)을 쓰면 새 VM 이 같은 IP 를 받을 수 있다.
kube-proxy 는 `iptables` 모드라 `strictARP` 는 신경 안 써도 된다(IPVS 에서만 필요).

### 더 큰 문제: 그 IP 는 NAT 안쪽

MetalLB IP 는 multipass NAT 안이라 **우분투에서만 닿는다.** 올려도 EXTERNAL-IP·Ingress ADDRESS·ArgoCD health 같은
**표시만 정상**이 되고, 맥·폰에서 앱에 닿는 길은 그대로 터널이다.

### 목표를 다시 보니 — 로컬 Railway

Railway 다움은 인터넷 공개가 아니라 사용 경험(push → 빌드·배포, 앱마다 주소)이다. 집 안 + VPN 으로 충분하고,
공개에 딸린 숙제(NetworkPolicy, 공인 TLS, 공격면)가 빠진다.

**클러스터 안쪽은 일반 k8s 와 똑같고, 다른 건 맨 바깥 입구를 어디에 여느냐뿐이다.**

| | 공개 서비스 | 로컬 Railway |
|---|---|---|
| 입구 | 공인 IP, 클라우드 LB | 집 LAN IP |
| DNS | 공인 도메인 | `*.home` |
| TLS | Let's Encrypt | 로컬 CA 또는 HTTP |
| 인그레스 아래 | 같음 | 같음 |

비표준인 건 "로컬"이 아니라 **VM 을 NAT 안에 둔 실습 구조**다. 대역 문제도, LAN 에서 안 보이는 것도 여기서 나온다.

Railway 까지 남은 큰 덩어리: ① 빌드(앱 저장소 push → 이미지 → 배포 저장소 태그 갱신) ② 앱 템플릿(공용 차트 + ApplicationSet)
③ LAN 에서 닿는 입구 + `*.home` DNS. MetalLB 보다 ③ 이 먼저다.

### NAT 안의 VM 으로 계속 갈 수 있나 — 세 가지 입구

| | 방식 | NAS 영향 | 배우는 것 | MetalLB |
|---|---|---|---|---|
| 1 | NAT 유지, 우분투 80 → VM NodePort 로 넘김 | 작음 | 적음 (우회) | 불필요 |
| 2 | VM 을 집 LAN 에 bridged 로 붙임 | **가장 큼** | 베어메탈 표준 구성 | **의미 있음** (집 LAN IP) |
| 3 | 우분투 본체에 k3s (원래 계획) | 중간 | k3s, 단일 노드 | 불필요 (ServiceLB) |

### bridged 가 NAS 에 주는 영향

- 적용 순간 수 초~수십 초 끊김: Jellyfin 재생, Samba 복사, Immich 백업, 그 시간의 restic 이 실패. 컨테이너는 안 죽는다
- **설정을 틀리면 원격으로 못 돌아온다.** SSH 경로도 같이 끊겨 모니터·키보드를 꽂아야 한다
- 브리지 MAC 이 바뀌면 우분투 IP 가 바뀔 수 있다 → Samba·Immich·공유기 포트포워딩·문서의 `192.168.45.4` 가 다 어긋남
- Docker 가 `FORWARD` 정책을 `DROP` 으로 바꿔 둬서 브리지에 붙인 VM 트래픽이 막힌다 → NAS 층 Docker 와 플랫폼이 얽힘

줄이는 법: 집에서 우분투 앞에 있을 때, `netplan try`(120초 후 자동 원복), 브리지에 고정 IP, restic·가족 사용 시간 피하기,
netplan·iptables 백업.

**주말에 우분투 앞에서 정하고 작업하기로.**

