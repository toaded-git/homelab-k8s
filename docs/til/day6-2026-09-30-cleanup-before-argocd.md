# Day 6 (2026-09-30) — ArgoCD 전에 치울 것들

← [Day 5](day5-2026-09-29-registry-and-deploy.md) · [목차](README.md)

이번 주 목표는 ArgoCD. 그 전에 짧은 것 세 개를 치운다.

1. Traefik Service 를 LoadBalancer → NodePort
2. 레지스트리 저장소를 emptyDir → PVC
3. 레플리카 2 + RWO 가 실제로 어떻게 깨지는지 실험

## 한 일

- Traefik 값 파일의 키를 `service.type` → `service.spec.type` 으로 고치고 `helm upgrade` (REVISION 2)
- 레지스트리 저장소를 emptyDir → PVC `registry-data`. 파드를 지워도 이미지가 남는 것 확인

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
