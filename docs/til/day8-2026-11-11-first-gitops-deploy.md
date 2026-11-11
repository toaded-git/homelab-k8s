# Day 8 (2026-11-11) — git push 로 첫 배포

← [Day 7](day7-2026-11-02-argocd-login.md) · [목차](README.md)

## 한 일

- 맥에서 매니페스트 한 줄 수정 → push → ArgoCD 감지 → sync. **cp1 에서 kubectl 없이** 클러스터를 바꿨다

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
