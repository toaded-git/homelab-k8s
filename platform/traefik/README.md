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

## 미해결: Service 가 아직 LoadBalancer

`kubectl -n traefik get svc` 의 EXTERNAL-IP 가 `<pending>` 이다.
맨땅 클러스터라 LoadBalancer 를 붙여 줄 것이 없기 때문. NodePort 30800 으로 접속되므로
동작에는 지장이 없지만, Ingress 의 ADDRESS 칸도 따라서 비어 있다.

원인: 이 차트(41.x)의 키는 `service.type` 이 아니라 **`service.spec.type`** 이다.
지금 `traefik-values.yaml` 은 없는 키를 설정하고 있어서 조용히 무시된다. 고칠 때는

```yaml
service:
  spec:
    type: NodePort
```

로 바꾸고 `helm upgrade traefik traefik/traefik --version 41.6.0 -n traefik -f traefik-values.yaml`.
