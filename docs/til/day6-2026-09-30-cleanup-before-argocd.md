# Day 6 (2026-09-30) — ArgoCD 전에 치울 것들

← [Day 5](day5-2026-09-29-registry-and-deploy.md) · [목차](README.md)

이번 주 목표는 ArgoCD. 그 전에 짧은 것 세 개를 치운다.

1. Traefik Service 를 LoadBalancer → NodePort
2. 레지스트리 저장소를 emptyDir → PVC
3. 레플리카 2 + RWO 가 실제로 어떻게 깨지는지 실험 → **실측 없이 개념 정리로 대체**

## 한 일

- Traefik 값 파일의 키를 `service.type` → `service.spec.type` 으로 고치고 `helm upgrade` (REVISION 2)
- 레지스트리 저장소를 emptyDir → PVC `registry-data`. 파드를 지워도 이미지가 남는 것 확인
- (저녁) ArgoCD 를 Helm 으로 설치 — 차트 10.9.4 / v3.5.3, NodePort 30443, Dex 끔

## Traefik: 틀린 키를 고치고, 이번엔 반영을 먼저 확인했다

어제 값 파일에 `service.type: NodePort` 를 넣었는데 Service 는 계속 LoadBalancer 였다.
차트 41.x 의 키는 `service.spec.type` 이다 ([Day 5](day5-2026-09-29-registry-and-deploy.md) 참고).
`helm get values` 에는 틀린 키도 **USER-SUPPLIED VALUES 로 멀쩡히 보인다.**
Helm 이 받아서 저장은 했지만 템플릿이 그 키를 읽지 않을 뿐이다.

반면 `ports.web.nodePort: 30800` 은 처음부터 먹혀 있었다. LoadBalancer 도 내부적으로
NodePort 를 할당하기 때문이다. 그래서 타입을 바꿔도 포트 번호는 그대로 간다.

### `helm upgrade` 의 모양

```
helm upgrade <릴리스> <차트> [플래그...]
```

```sh
helm upgrade traefik traefik/traefik \
  --version 41.6.0 \
  -n traefik -f traefik-values.yaml \
  --dry-run=server
```

| 부분 | 빼면 |
|---|---|
| `-n traefik` | `default` 에서 릴리스를 찾다가 `has no deployed releases` |
| `--version` | 그 사이 나온 최신 차트로 올라간다. 재현성이 깨진다 |
| `-f` | 값 파일은 차트 자리가 아니라 이 플래그 뒤. 상대 경로라 파일 있는 곳에서 친다 |
| `--dry-run=server` | 확인이 끝나면 빼고 다시 실행 |

명령이 길다. 차트·버전·값·네임스페이스를 **매번 손으로 말해야 해서**다.
ArgoCD 로 가면 이걸 git 에 선언해 두고 컨트롤러가 적용한다. 긴 명령 한 줄이 파일 하나가 된다.

### 결과

| | 전 | 후 |
|---|---|---|
| REVISION | 1 | 2 |
| TYPE | LoadBalancer | **NodePort** |
| EXTERNAL-IP | `<pending>` | `<none>` |
| PORT(S) | `80:30800`, `443:30843` | 그대로 |
| ClusterIP | `10.98.237.66` | 그대로 — Service 를 새로 만든 게 아니라 고친 것 |
| Traefik 파드 | | 재시작 없음 — 바뀐 건 Service 뿐 |

`Host: dashboard.home` → 200, 다른 호스트 → 404.

### Ingress 의 ADDRESS 는 여전히 비어 있다

"LoadBalancer 가 pending 이라 ADDRESS 가 빈다"고 생각했는데, NodePort 로 바꿔도 비어 있었다.

```
$ kubectl get ingress dashboard -o jsonpath='{.status}'
{"loadBalancer":{}}
```

Traefik 은 ADDRESS 를 스스로 정하지 않는다. **자기 Service 의 `status.loadBalancer` 를
Ingress status 로 옮겨 적을 뿐이다.** 그 값을 채우는 건 클라우드의 LB 컨트롤러나 MetalLB 다.

- 전: LoadBalancer 인데 채워 줄 주체가 없음 → `<pending>`, ADDRESS 빈칸
- 후: NodePort 라 채울 게 없음 → `<none>`, ADDRESS 빈칸

원인은 타입이 아니라 **LB 상태를 채워 주는 주체가 없다**는 것이었다.
채우려면 MetalLB 를 올리거나(k3s 에선 ServiceLB 가 이 역할) Traefik 에 보고할 주소를 직접 준다.
동작에는 영향이 없어서 두었다.

## 레지스트리: emptyDir → PVC

바꾼 건 `volumes` 한 줄이다. `registry:2` 는 이미지를 `/var/lib/registry` 에 두는데,
`mountPath` 는 처음부터 그 경로였다. 컨테이너 쪽은 그대로 두고 **거기에 무엇을 붙이느냐**만 바꿨다.

```yaml
volumes:
- name: data
  persistentVolumeClaim:        # emptyDir: {} 였던 자리
    claimName: registry-data
```

### 적용 순서가 거꾸로였다

Deployment 를 PVC 보다 먼저 올렸다.

```
Warning  FailedScheduling  0/3 nodes are available: persistentvolumeclaim "registry-data" not found
Normal   Scheduled         Successfully assigned default/registry-... to w1
```

없는 PVC 를 가리키니 스케줄러가 못 놓다가, 몇 초 뒤 PVC 가 생기자 재시도해서 붙었다.
쿠버네티스는 "지금 없으면 계속 다시 해 본다"는 방식이라 순서가 틀려도 결국 맞춰진다.
그래도 Warning 이 남으니 **PVC → Deployment** 순서로 올린다. ArgoCD 에서는 sync wave 가 이 순서를 정한다.

### 새 PVC 는 비어 있다

바꾸는 순간 `_catalog` 가 `[]` 가 됐다. 예전 이미지는 사라진 emptyDir 에 있었으니 당연하다.
떠 있던 대시보드 파드는 노드에 이미지가 캐시돼 있어서 영향이 없었고, 우분투에서 다시 push 했다.

### 확인: 파드를 지워도 남는가

```sh
kubectl delete pod -l app=registry     # -l: 이름 대신 라벨로 고른다. 해시가 붙은 이름을 찾을 필요 없음
curl http://10.23.218.234:30500/v2/_catalog
# {"repositories":["homelab-dashboard"]}
curl http://10.23.218.234:30500/v2/homelab-dashboard/tags/list
# {"name":"homelab-dashboard","tags":["v1"]}
```

`_catalog` 는 레지스트리 API(OCI Distribution)의 저장소 목록 엔드포인트다. 레지스트리가 `/var/lib/registry` 를
훑어서 답하므로, 파드가 바뀌어도 목록이 남아 있다면 데이터가 파드 밖에 있다는 증거가 된다.

### 파드 이름의 해시로 무슨 일이 있었는지 읽기

| 사건 | 파드 이름 | 의미 |
|---|---|---|
| apply 전 | `registry-5b6799f747-2glfq` | |
| apply 후 | `registry-8645ff9ff5-6ppjj` | **가운데 해시가 바뀜** — 템플릿이 바뀌어 ReplicaSet 이 새로 생김 |
| 파드 삭제 후 | `registry-8645ff9ff5-fmxz7` | 가운데 그대로, 끝만 바뀜 — 같은 ReplicaSet 이 채워 넣음 |

새 파드는 **다시 w1** 에 떴다. 우연이 아니다. `WaitForFirstConsumer` 라 PV 는 파드가 처음 뜬 w1 의 디스크에
만들어졌고, PV 에 nodeAffinity 가 붙어서 스케줄러는 w1 말고는 고를 수 없다.
NodePort 라 접속 주소(cp1 IP)는 그대로 써도 된다. 대신 w1 이 죽으면 레지스트리도 멈춘다.

## RWO: 실험 대신 개념 정리

ArgoCD 의 선행 작업이 아니라서 실측은 건너뛰었다. 아래 "예상"은 확인하지 않은 것이다.

### RWO 는 "파드 하나"가 아니라 "노드 하나"

접근 모드는 PVC 에 적는 **붙는 규칙**이다. 뒤에 무슨 저장소가 있는지는 StorageClass 가 정한다.

| | 같은 노드의 파드 여러 개 | 다른 노드의 파드 |
|---|---|---|
| RWO (ReadWriteOnce) | 같이 붙음 | 못 붙음 |
| RWOP (ReadWriteOncePod) | 파드 하나만 | 못 붙음 |
| RWX (ReadWriteMany) | 같이 붙음 | 같이 붙음 |

RWO 는 USB 외장 디스크, RWX 는 NFS·Samba 공유 폴더에 가깝다.
w1·w2 에 나뉜 파드가 **같은 RWO 볼륨을 같이 보는 일은 없다.**

### 다른 노드에 두 번째 파드를 띄우려 하면

- **local-path (지금):** 볼륨이 w1 디스크의 폴더 그 자체라 w2 는 닿을 수 없다. PV 의 nodeAffinity 때문에
  스케줄러가 처음부터 w1 에만 놓는다. 예상: **둘 다 w1 에 떠서 "되는 것처럼" 보인다.**
  노드 장애 분산은 안 되고, 앱이 같은 파일을 동시에 쓰면 망가질 수 있는데 쿠버네티스는 그걸 막지 않는다
- **네트워크 디스크 (EBS 등):** 어느 노드든 붙을 수는 있지만 한 번에 하나. 다른 노드에 놓인 파드는
  `Multi-Attach error` 로 `ContainerCreating` 에서 멈춘다. 흔히 말하는 "RWO 가 깨진다"는 이쪽
- podAntiAffinity 로 파드를 다른 노드에 두라고 강제하면, 갈 곳이 없어 두 번째 파드가 **Pending**

### 그럼 이중화하는 웹 서비스의 파일 저장은

**파일은 오브젝트 스토리지(S3 호환)로 빼고 파드는 무상태로 둔다.** 세션을 Redis 로 빼는 것과 같은 이유.

- 볼륨은 파드 안에 **디렉터리로 마운트**된다. S3 는 마운트가 아니라 앱이 **HTTP API 로 호출**하는 외부 서비스라
  접근 모드라는 개념이 없고, 파드가 몇 개든 어느 노드든 상관없다
- RWX(NFS 등)도 되지만 파일 잠금·성능·단일 장애점 문제가 있어 레거시 앱을 그대로 옮길 때 쓴다
- 클러스터 안에 S3 서버를 올리면 **그 서버가 RWO 볼륨에 저장한다.** 상태 있는 부분을 한 곳에 모으는 구조

```
앱 파드들 (무상태) ──HTTP──> S3 서버 파드 ──마운트──> RWO 볼륨 (노드 디스크)
```

대시보드는 `/data` 에 로컬 파일을 쓰므로 지금 구조에선 레플리카 1 이 맞다.

## ArgoCD 설치 (저녁)

### `helm repo add` 에는 소스 저장소가 아니라 차트 저장소를

처음엔 `helm repo add argoCD https://github.com/argoproj/argo-cd` 로 시작하려 했다. 그건 **소스 코드 저장소**다.
Helm 저장소에는 `index.yaml`(차트 목록)이 있어야 한다. Maven 저장소의 `maven-metadata.xml` 같은 것.

| 주소 | `index.yaml` |
|---|---|
| `github.com/argoproj/argo-cd` | 404 |
| `argoproj.github.io/argo-helm` | 200 |

Traefik 도 소스는 `github.com/traefik/traefik`, 차트는 `traefik.github.io/charts` 였다.

### 차트 버전 ≠ 앱 버전

```
NAME              CHART VERSION   APP VERSION
argo-cd/argo-cd   10.9.4          v3.5.3
```

`--version` 에는 차트 버전. 차트의 `kubeVersion: '>=1.25.0-0'` 은 하한선일 뿐 1.37 에서 테스트됐다는 뜻은 아니다.

### values 는 외우는 게 아니라 찾아 읽는 것

Traefik 값 파일을 그대로 가져다 쓰려다 막혔다. `kubectl explain` 은 **쿠버네티스 API 스키마**라 어디서나 같지만,
`helm show values` 는 **차트 작성자가 만든 설정 인터페이스**라 차트마다 다르다. Spring 의 `server.port` 가
다른 프레임워크에선 다른 이름인 것과 같다.

읽는 요령:

```sh
helm show values argo-cd/argo-cd --version 10.9.4 | grep -nE '^[a-zA-Z-]+:'   # 최상위 키만
```

ArgoCD 차트의 최상위 키는 대부분 **구성 요소 하나씩**이다 — `controller`(상태 비교·동기화), `repoServer`(git clone·렌더링),
`server`(API·UI), `dex`(SSO), `redis`/`redis-ha`(캐시). 나머지(`global`, `configs`, `crds`)는 공통 설정.
들여쓰기를 점으로 이으면 키의 주소가 된다: `server → service → type` = `server.service.type`.
**값 파일에는 기본값과 다른 것만 적는다.**

솔직히 이 단계는 감이 안 와서 값 파일을 설명 듣고 받아 적었다. 결과는 여섯 줄이었다.

```yaml
server:
  service:
    type: NodePort
    nodePortHttp: 30880   # 기본 30080 이 대시보드 NodePort 와 충돌
    nodePortHttps: 30443
dex:
  enabled: false          # SSO 안 씀
```

**기본값이 내 클러스터와 충돌하는 지점**이 또 나왔다. 그대로 설치했으면 `provided port is already allocated`.

### 첫 설치가 `context canceled` 로 실패

```
STATUS: failed
DESCRIPTION: Release "argocd" failed: context canceled
```

Helm 클라이언트가 도중에 취소 신호를 받은 것(Ctrl+C, 세션 끊김 등). CRD 조차 없어서 아무것도 올라가기 전에 끊겼다.
실패 기록이 남아 같은 이름으로 `install` 이 안 되므로 `helm uninstall` 후 재설치. 설치는 1~2분 걸리니 기다릴 것.

또 하나 — `kubectl get pods` 에 안 보인 건 `-n argocd` 를 빼서다. 기본은 `default` 네임스페이스.

### 결과

| | |
|---|---|
| 파드 | 6개 Running (application-controller, applicationset-controller, notifications, redis, repo-server, server). Dex 없음 |
| `argocd-server` | NodePort `80:30880`, `443:30443`. http 는 https 로 307 |
| CRD | `applications`, `applicationsets`, `appprojects` `.argoproj.io` |

`argocd-application-controller-0` 만 해시 대신 `-0` 이 붙는다. **StatefulSet** 이라서. 여러 개 띄우면 번호별로
담당 클러스터를 나누는 구조라 순번 이름이 필요하다.

