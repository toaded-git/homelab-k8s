# Day 5 (2026-09-29) — 사설 레지스트리와 첫 배포, 그리고 디버깅

← [Day 4](day4-2026-09-28-service-and-image.md) · [목차](README.md)

## 한 일

- 우분투 Docker 에 사설 레지스트리 등록 → `docker push` 성공
- VM 3대 containerd 에 같은 레지스트리 등록
- 대시보드 Deployment(레플리카 2) + Service(NodePort 30080) 작성·배포
- CrashLoopBackOff 진단 → ConfigMap 으로 설정 주입 → 기동 성공

## 막힌 곳

### 평문 HTTP 레지스트리 — 양쪽에 예외가 필요했다

기본적으로 HTTPS 를 강제한다. 안 하면 `http: server gave HTTP response to HTTPS client`.

| 누가 | 방향 | 어디에 |
|---|---|---|
| 호스트 Docker | **push** | `daemon.json` 의 `insecure-registries` |
| VM 3대 containerd | **pull** | `certs.d/<호스트:포트>/hosts.toml` |

Docker 는 한 줄이면 되는데, **containerd 2.x 는 그런 옵션이 없다.**
`config.toml` 에서 디렉터리를 가리키고, 그 안에 호스트별 파일을 두는 방식이다.

```toml
[plugins.'io.containerd.cri.v1.images'.registry]
  config_path = '/etc/containerd/certs.d'
```

```toml
# /etc/containerd/certs.d/10.23.218.234:30500/hosts.toml
server = "http://10.23.218.234:30500"

[host."http://10.23.218.234:30500"]
  capabilities = ["pull", "resolve"]
```

**containerd 1.x 는 `io.containerd.grpc.v1.cri` 라 키 경로가 다르다.**
검색으로 나오는 예제 대부분이 1.x 기준이어서 그대로 쓰면 조용히 무시된다.

`config_path` 는 기동 시에만 읽으므로 containerd 재시작이 필요하다.
대신 그 뒤로 `certs.d` 안에 파일을 더 넣는 건 재시작 없이 먹는다.

### `config_path = ''` 가 파일에 두 군데 있었다

이미지 레지스트리(54행)와 transfer 플러그인(245행). 그냥 치환하면 무관한 데까지 바뀐다. 줄 번호를 지정했다.

### `daemon.json` 에 이미 nvidia 런타임 설정이 있었다

덮어썼으면 Jellyfin·Immich 의 GPU 트랜스코딩이 죽는다. 병합하고 백업을 떠 뒀다.
`live-restore: true` 도 같이 켰지만, **켜는 그 재시작 자체는 적용 전이라 컨테이너가 한 번 재시작된다.**

### YAML 들여쓰기 한 칸 때문에 apply 가 거부됐다

```yaml
volumes:
- name: config
  configMap:              # 값이 비어 있고
  name: dashboard-config  # 형제로 들어가서 볼륨 이름을 덮어써 버림
  resources: {}           # 컨테이너에서 딸려온 잔재
```

```
strict decoding error: unknown field "spec.template.spec.volumes[0].resources"
```

`name` 이 `configMap` 과 같은 깊이라 **볼륨 자신의 필드**가 되어, 볼륨 이름이 `config` → `dashboard-config` 로 바뀌었다.
그러면 `volumeMounts` 의 `name: config` 와 짝이 안 맞는다.

### 워커에서 `kubectl` 이 안 됐다

```
The connection to the server localhost:8080 was refused
```

워커에는 kubeconfig 가 없다. `kubeadm join` 은 kubelet 인증서만 받고 관리자 자격증명은 안 받는다.
`kubeadm init` 이 만드는 `/etc/kubernetes/admin.conf` 는 **컨트롤 플레인에만** 있다.
kubeconfig 를 못 찾으면 kubectl 은 아주 옛날 기본값인 `localhost:8080` 으로 붙으려 한다.

→ 컨트롤 플레인에서 쓰면 된다. 다른 데서 쓰려면 별도 사용자 인증서 + RBAC 이 맞는 길이다.
(admin.conf 를 노드마다 복사하는 건 전권 자격증명을 뿌리는 짓)

### 그리고 vim 이 마우스를 가로챘다

PuTTY 에서 우클릭 붙여넣기가 비주얼 모드 선택으로 먹혔다. `:set mouse=` 로 해결.
YAML 붙여넣을 때는 `:set paste` 도 필요하다 — 안 걸면 자동 들여쓰기가 누적돼서 계단처럼 밀린다.
**들여쓰기가 문법인 포맷에서는 치명적.**

## 배운 것

### `ImagePullBackOff` 와 `CrashLoopBackOff` 는 완전히 다른 신호다

- `ImagePullBackOff` — 이미지를 못 가져왔다 (레지스트리·인증·태그 문제)
- `CrashLoopBackOff` — **가져와서 실행까진 됐는데 죽었다** (앱 문제)

즉 CrashLoop 이 떴다는 건 레지스트리 설정이 성공했다는 뜻이기도 하다. 좌절할 에러가 아니라 전진의 신호였다.

### `logs` 와 `describe` 는 보는 층이 다르다

| 명령 | 관점 | 볼 것 |
|---|---|---|
| `kubectl logs` | **앱** — 컨테이너 안 프로세스의 stdout/stderr | 에러 메시지 원문. 직전 죽은 것은 `--previous` |
| `kubectl describe pod` | **쿠버네티스** | 맨 아래 `Events`, `Last State`, `Exit Code` |

Exit Code 읽는 법: `1` 앱이 스스로 에러 내고 종료 / `137` SIGKILL (OOM 이나 liveness 실패) / `139` segfault.

`describe` 의 `Mounts:` 를 보면 **의도한 볼륨이 실제로 붙었는지**가 드러난다.
serviceaccount 토큰만 있으면 내 볼륨은 하나도 안 붙은 것이다.

### CrashLoop 은 지수 백오프

재시도 간격이 10초 → 20초 → 40초 … 최대 5분까지 벌어진다. 고친 뒤 반응이 느린 건 이 때문.

### `volumes` 와 `volumeMounts` 는 짝이다

- `volumes` — 파드 레벨. **무엇을** 쓸지 (ConfigMap? Secret? PVC?)
- `volumeMounts` — 컨테이너 레벨. **어디에** 붙일지
- `name` 으로 짝지어진다. 파드 안에서만 통하는 별명이고, ConfigMap 실제 이름과는 별개다

Docker Compose 의 최상위 `volumes:` 선언과 서비스별 `volumes:` 항목 관계와 같다.

### 같은 경로에 볼륨 두 개를 마운트할 수 없다

앱이 `/config/config.yaml`(설정)과 `/config/id_ed25519`(개인키)를 같은 디렉터리에서 기대했는데,
전자는 ConfigMap, 후자는 Secret 이라 **둘 다 `/config` 에 걸 수가 없다.**

→ 앱 설정의 키 경로를 `/secrets/` 로 바꿔서 분리했다.
(대안: projected 볼륨으로 ConfigMap + Secret 을 한 볼륨에 합치기. 남의 이미지라 경로를 못 바꿀 때 쓸 카드)

### Secret 은 암호화가 아니다

base64 인코딩일 뿐이다. `kubectl get secret -o yaml` 하면 누구나 디코드하고, etcd 안에서도 기본은 평문이다.
ConfigMap 과의 실질적 차이는 RBAC 으로 따로 권한을 걸 수 있다는 것, describe·로그에 값이 안 찍힌다는 것 정도.
**그래서 Secret YAML 은 저장소에 커밋하면 안 된다.**

### `--dry-run=client` 와 `--dry-run=server` 는 다르다

- `client` — 서버에 안 보내고 YAML 만 찍는다. **스키마 오류를 못 잡는다**
- `server` — apiserver 가 검증만 하고 저장하지 않는다. 필드 오타·구조 오류를 잡아준다

손으로 고친 매니페스트는 **server** 로 한 번 거치는 습관이 낫다.

### `kubectl explain`

```sh
kubectl explain deployment.spec.template.spec.volumes.configMap
```

문서를 뒤지지 않고 **API 서버 스키마에서** 필드 구조를 바로 꺼낼 수 있다. 버전이 정확히 맞는 문서라서 더 믿을 만하다.

### 파드 템플릿이 바뀌면 ReplicaSet 이 새로 생긴다

파드 이름의 해시가 `5449ffd9b` → `75d77ff98` 로 바뀌는 것으로 확인된다.
Deployment 는 새 ReplicaSet 을 만들고 롤링 업데이트를 돌리며, **옛 ReplicaSet 은 롤백용으로 남는다.**

### 좋은 앱은 부분 실패로 죽지 않는다

설정을 못 읽었을 때는 `ERROR` 후 종료(Exit 1)했지만, SSH 키를 못 읽었을 때는 `WARN` 만 남기고 웹 서버는 계속 떴다.

```
level=INFO  msg=listening addr=:8080 hosts=3
level=WARN  msg="collect failed" host=ubuntu err="open /secrets/id_ed25519: no such file or directory"
```

**에러의 심각도가 내려간 것으로 진행 상황을 읽을 수 있었다.** 이게 로그를 잘 설계한 앱의 값어치다.

---
← [Day 4](day4-2026-09-28-service-and-image.md) · [목차](README.md)

---

## 덧 — 같은 날 오후: 저장소까지

Secret 과 PVC 를 마저 붙여 앱을 끝까지 띄웠다.

### `WaitForFirstConsumer` 를 타임스탬프로 봤다

kubeadm 은 StorageClass 를 안 깔아준다 (`kubectl get sc` → `No resources found`).
클라우드면 CSI 드라이버가 기본 StorageClass 를 주지만 맨땅 클러스터는 직접 마련해야 한다.
Rancher `local-path-provisioner` v0.0.37 을 설치했다. **k3s 가 기본 탑재하는 것**이라 다음 단계와도 이어진다.

```
PVC 나이: 18분      ← 만들자마자 Pending 으로 대기
PV  나이: 2분 40초  ← 파드를 apply 한 순간 생성
```

로컬 디스크는 "어느 노드"인지가 곧 정체성인데, PVC 를 만드는 시점엔 파드가 어디 갈지 모른다.
먼저 볼륨을 만들면 파드를 그 노드에 묶게 되므로, **스케줄러가 노드를 정한 뒤에 볼륨을 만든다.**
그래서 PVC 가 `Pending` 인 것은 에러가 아니다.

주의: 이 StorageClass 는 **기본값이 아니라서** PVC 에 `storageClassName: local-path` 를 명시해야 하고,
`reclaimPolicy: Delete` 라 **PVC 를 지우면 데이터도 지워진다.**

### `explain` 은 예제 생성기가 아니다

PVC 는 `kubectl create` 하위 명령이 없어서 손으로 써야 하는데, 여기서 한참 헤맸다.

`kubectl explain pvc.spec` 의 출력에는 `requests` 가 없다. 타입이 `<VolumeResourceRequirements>` 로 적힌
필드는 **덩어리**라서 한 단계 더 들어가야(`explain pvc.spec.resources`) 하위 필드가 보인다.
즉 **`explain` 은 물어본 위치의 자식만 보여준다.** `apiVersion`·`kind`·`metadata` 가 안 보였던 것도
그것들은 뿌리(`explain pvc`)에 있기 때문이다.

- `<string>` / `<[]string>` → 값을 바로 쓴다
- 대문자 타입 이름 → 더 파고들어야 한다
- `-required-` → 필수. Deployment spec 은 `selector` 와 `template` 둘뿐이다

`kubectl explain pvc --recursive` 로 트리 전체를 한 번에 펼치는 게 실전에서는 더 편하다.

그리고 **스키마상 필수와 실제로 필요한 것은 다르다.** PVC spec 에는 `-required-` 가 하나도 없지만
`accessModes` 없이는 거부당하고, `storageClassName` 없이는 영원히 Pending 이다 (에러도 안 난다).

### 결과

```
level=INFO msg="history persisted" path=/data/history.db retention=168h0m0s
level=INFO msg=listening addr=:8080 hosts=3
level=INFO msg="ssh connected" host=ubuntu  addr=192.168.45.4:22
level=INFO msg="ssh connected" host=macmini addr=192.168.45.11:22
```

파드(10.244.x) → VM NAT → 집 LAN 으로 SSH 가 붙었다. ufw 는 막지 않았다.
`curl http://10.23.218.234:30080/api/health` → `{"status":"ok"}`.

**설정·비밀값·저장소를 모두 바깥에서 주입하는 파드**가 완성됐다. 이미지에는 아무것도 구워져 있지 않다.

---

## 덧 2 — 같은 날 저녁: 인그레스

### probe 는 readiness 와 liveness 의 역할이 다르다

| | 질문 | 실패하면 |
|---|---|---|
| `readinessProbe` | 지금 트래픽 받을 수 있나 | Service 대상에서 **제외** (재시작 안 함) |
| `livenessProbe` | 죽었나 | 컨테이너 **재시작** |

**외부 의존성(DB 등)은 readiness 에만 넣어야 한다.** liveness 에 넣으면 DB 가 죽었을 때
앱을 계속 재시작하는데, 재시작해도 DB 는 살아나지 않아 무한 루프가 된다.
liveness 는 "죽였다 살리면 나아지나?"만 봐야 한다.

Spring Boot 는 actuator 가 `/actuator/health/liveness`·`/readiness` 를 나눠서 준다.
JVM 처럼 기동이 느리면 `startupProbe` 로 기동 구간을 분리한다 — 통과 전까지 다른 probe 는 돌지 않으므로
**기동은 느긋하게, 운영은 예민하게** 둘 다 가능하다.

리소스도 같이 지정해 QoS 가 `BestEffort` → `Burstable` 이 됐다.
CPU 초과는 쓰로틀링(느려짐)이고 메모리 초과는 OOMKill(`Exit Code 137`)이라, 메모리 limit 은 넉넉히 잡는다.

### ingress-nginx 가 은퇴했다

설치하려고 확인해 보니 `kubernetes/ingress-nginx` 저장소가 **아카이브 상태**였다
(마지막 릴리스 2026-03-19). 보안 패치가 더 나오지 않는다.

살아 있는 것 중 **Traefik** 을 골랐다. Ingress 와 Gateway API 를 둘 다 지원해서,
지금은 Ingress 로 배우고 나중에 같은 컨트롤러로 Gateway API 를 비교할 수 있기 때문.

**Ingress 리소스 YAML 은 컨트롤러가 달라도 거의 같다.** 달라지는 건 어노테이션뿐이라,
Traefik 으로 배워도 ingress-nginx 를 쓰는 곳에서 바로 통한다.

### Helm 은 값의 경로가 곧 문법이다

`service.type: NodePort` 를 값 파일에 넣었는데 Service 가 계속 LoadBalancer 였다.
차트 41.x 의 실제 키는 **`service.spec.type`** 이었다. **없는 키를 설정하면 오류 없이 무시된다.**

containerd 1.x/2.x 의 플러그인 경로와 같은 함정이다. 설치 전에 반드시

```sh
helm show values traefik/traefik --version 41.6.0
```

로 실제 구조를 확인할 것. 그리고 `helm install --dry-run --debug` 로 펼쳐진 매니페스트를 한 번 본다.

### Ingress 는 프록시가 아니라 규칙이다

Ingress 리소스를 만들어도 **새 파드가 생기지 않는다.** 이미 떠 있는 Traefik 파드에
라우팅 규칙 한 줄이 등록될 뿐이다. nginx 의 `server` 블록 하나를 추가하는 것과 같고,
다른 점은 설정 파일을 고쳐 reload 하는 대신 컨트롤러가 API 서버를 보고 있다가 스스로 갱신한다는 것.

```
curl -H "Host: dashboard.home" → 노드:30800
  → NodePort(iptables, L4) → Service traefik
  → Traefik 파드 ← 여기서 Host 헤더를 읽고 규칙 조회 (L7)
  → Service dashboard(ClusterIP) → 파드 :8080
```

Service 를 없애는 게 아니라 **앞에 L7 한 겹을 얹는 것**이다. Service 의 로드밸런싱은
커널 iptables 의 확률 분기라 IP·포트만 보지만, Traefik 은 프로세스가 HTTP 를 파싱하므로
Host·경로·헤더로 가를 수 있고 TLS 종료도 한다.

검증에서 **404 가 더 중요한 확인이었다.** 등록하지 않은 호스트로 보냈을 때 404 가 왔다는 건
요청이 Traefik 까지 도달했고 규칙에 없어서 거절했다는 뜻이다. 연결 자체가 실패했으면
Connection refused 나 타임아웃이 났을 것이다.

### 그리고 이 반복이 PaaS 를 만드는 이유

앱 하나에 Deployment · Service · Ingress(+ ConfigMap · Secret · PVC). 앱이 늘 때마다
같은 모양을 복사해 이름만 바꾼다. 이 반복을 없애는 게 Helm 차트이고, 그걸 자동으로 찍어내는 게
Railway · Heroku 가 하는 일이다. **손으로 다 해 본 지금이 무엇을 자동화할지 아는 자리다.**
