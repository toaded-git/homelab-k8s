# argocd

GitOps 컨트롤러. Helm 으로 설치했다.

```sh
helm repo add argo-cd https://argoproj.github.io/argo-helm
helm install argocd argo-cd/argo-cd \
  --version 10.9.4 \
  -n argocd --create-namespace \
  -f argocd-values.yaml
```

- 차트 `argo-cd-10.9.4`, 앱 `v3.5.3` (차트 `kubeVersion: >=1.25`. 1.37 이 공식 테스트 대상인지는 미확인)
- 저장소 별칭을 `argo-cd` 로 붙였다. 공식 예제는 `argo` 를 쓰므로 `argo/argo-cd` 로 나온다
- 릴리스·네임스페이스를 `argocd` 로 맞춰야 파드 이름(`argocd-server` 등)이 문서와 같아진다
- `github.com/argoproj/argo-cd` 는 소스 저장소라 `index.yaml` 이 없다(404). 차트 저장소는 `argo-helm`

## 값 파일

| 값 | 이유 |
|---|---|
| `server.service.type: NodePort` | 기본 ClusterIP. 웹 UI 를 밖에서 보려고 |
| `server.service.nodePortHttp: 30880` | **기본 30080 이 대시보드 NodePort 와 충돌** |
| `server.service.nodePortHttps: 30443` | 기본값과 같지만 쓰는 포트를 한눈에 보려고 |
| `dex.enabled: false` | SSO 안 씀. admin 계정으로 로그인 |
| (redis-ha) | 기본값이 이미 꺼짐 → 적지 않음 |

## 접속

```sh
# 맥 (집 안). 밖이면 -J uuook@<외부IP>:<포트> 를 앞에 붙인다
ssh uuook@192.168.45.4 -L 30443:10.23.218.234:30443
# 브라우저: https://localhost:30443 (자체 서명 인증서 경고는 정상)
```

http 30880 은 https 로 307 리다이렉트된다. argocd-server 가 TLS 를 직접 하기 때문.
Traefik Ingress 뒤에 두려면 `configs.params."server.insecure": true` 로 TLS 를 꺼야 리다이렉트가 반복되지 않는다.

초기 비밀번호:

```sh
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
```

로그인 후 비밀번호를 바꾸고 이 Secret 은 지운다.
