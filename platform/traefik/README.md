# traefik

인그레스 컨트롤러. Helm 으로 설치했다.

```sh
helm repo add traefik https://traefik.github.io/charts
helm install traefik traefik/traefik \
  --version 41.6.0 \
  --namespace traefik --create-namespace \
  -f traefik-values.yaml
```

- 차트 `traefik-41.6.0`, 앱 `v3.7.13` (요구 k8s >= 1.25)
- IngressClass `traefik` 가 클러스터 기본값으로 생긴다
- 진입 포트: HTTP NodePort **30800**, HTTPS **30843**

## 왜 ingress-nginx 가 아닌가

`kubernetes/ingress-nginx` 는 **2026-03 에 아카이브**됐다 (마지막 릴리스 2026-03-19).
보안 패치가 더 나오지 않는다. 살아 있는 것 중 Traefik 만 **Ingress 와 Gateway API 를 둘 다**
지원해서, 나중에 같은 컨트롤러로 Gateway API 실습까지 이어갈 수 있다.

Ingress 리소스 YAML 은 컨트롤러가 달라도 거의 같다. 달라지는 건 어노테이션뿐.

## Service 타입: LoadBalancer → NodePort (2026-09-30 해결)

처음 설치했을 때는 값 파일에 `service.type: NodePort` 를 넣었는데도 Service 가 `LoadBalancer`
로 떠서 EXTERNAL-IP 가 `<pending>` 이었다. 이 차트(41.x)의 키는 `service.type` 이 아니라
**`service.spec.type`** 이다. 없는 키를 넣어도 Helm 은 에러 없이 무시한다.

`helm show values traefik/traefik --version 41.6.0` 에서 구조를 확인하고 값 파일을 고친 뒤,
dry-run 으로 반영을 먼저 확인하고 올렸다.

```sh
helm upgrade traefik traefik/traefik \
  --version 41.6.0 \
  -n traefik -f traefik-values.yaml \
  --dry-run=server          # 출력의 Service type·nodePort 확인 후 이 줄을 빼고 실행
```

결과: 릴리스 REVISION 2, Service `NodePort`, `80:30800`·`443:30843` 유지, ClusterIP 유지,
Traefik 파드는 재시작하지 않음.

### Ingress 의 ADDRESS 는 여전히 비어 있다 (정상)

Traefik 은 **자기 Service 의 `status.loadBalancer`** 를 Ingress status 로 옮겨 적는다.
그 값을 채우는 건 클라우드 LB 컨트롤러나 MetalLB 같은 것이라, 맨땅 클러스터에서는
LoadBalancer 든 NodePort 든 빈칸이다. 타입 때문이 아니다.

채우고 싶으면 MetalLB 를 올려 LoadBalancer 로 되돌리거나, Traefik 에 보고할 주소를
직접 지정한다(`ingressEndpoint`). 동작에는 영향이 없어서 지금은 두지 않는다.
