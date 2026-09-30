# Day 6 (2026-09-30) — ArgoCD 전에 치울 것들

← [Day 5](day5-2026-09-29-registry-and-deploy.md) · [목차](README.md)

이번 주 목표는 ArgoCD. 그 전에 짧은 것 세 개를 치운다.

1. Traefik Service 를 LoadBalancer → NodePort
2. 레지스트리 저장소를 emptyDir → PVC
3. 레플리카 2 + RWO 가 실제로 어떻게 깨지는지 실험

## 한 일

- Traefik 값 파일의 키를 `service.type` → `service.spec.type` 으로 고치고 `helm upgrade` (REVISION 2)

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
