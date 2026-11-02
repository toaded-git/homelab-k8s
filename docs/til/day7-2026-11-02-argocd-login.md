# Day 7 (2026-11-02) — 한 달 만에 돌아와서

← [Day 6](day6-2026-09-30-cleanup-before-argocd.md) · [목차](README.md)

9/30 에 ArgoCD 를 설치하고 한 달 넘게 손을 안 댔다. 그 사이 클러스터에 무슨 일이 있었는지부터 봤다.

## 한 일

- 방치된 클러스터 점검 (읽기 전용)
- ArgoCD UI 로그인 — 밖에서 SSH 점프 + 터널
- admin 비밀번호 변경, 초기 Secret 삭제
- ArgoCD 용 git 읽기 전용 SSH 키 — `authorized_keys` 로 제한 (한 번 틀리고 고침)

## 방치된 클러스터 점검

노드 3대 Ready, ArgoCD 파드 6개 Running, UI 200. 겉보기엔 멀쩡했다. 그런데 재시작 횟수가 이상했다.

```
argocd-application-controller-0   Running   1 (17d ago)
argocd-repo-server-...            Running   8 (9d ago)
```

### 1. VM 이 통째로 재시작돼 있었다

우분투 본체 `uptime` 35일, VM `w1` 은 17일. **호스트는 그대로인데 VM 만 10월 중순에 재시작됐다.**
거의 모든 파드의 재시작이 1 인 이유. 클러스터는 9/27 호스트 재부팅 때처럼 알아서 복구됐다.
원인(multipass 데몬 재시작, snap 자동 갱신 등)은 아직 확인 안 함.

### 2. repo-server 는 liveness 가 죽였다

```sh
kubectl get pod <repo-server> -o jsonpath='{.status.containerStatuses[0].lastState}'
# {"terminated":{"exitCode":0,"reason":"Completed", ...}}
kubectl logs <repo-server> --previous
```

```
level=error msg="Error serving health check request" duration=5.000s error="context canceled"
level=error msg="Error serving health check request" duration=5.002s error="context canceled"
level=info  msg="got signal terminated, attempting graceful shutdown"
```

헬스 체크가 5초 타임아웃 안에 답을 못 했고, liveness 실패가 쌓이자 kubelet 이 SIGTERM 을 보냈다.
**OOMKilled 가 아니다** — 종료 사유 `Completed`, exit 0. 메모리 한도도 설정돼 있지 않았다.

[Day 5](day5-2026-09-29-registry-and-deploy.md) 에서 대시보드 probe 를 정하며 "liveness 를 조이면 멀쩡한 파드를 죽인다"고
적었는데, 그 현상을 남이 만든 차트 기본값에서 봤다. CPU 를 VM 3대와 NAS 가 나눠 쓰니 순간적으로 느려졌을 수 있다(추측).
마지막이 10-23 이고 그 뒤 멀쩡해서 지켜보기로. 재발하면 probe 타임아웃을 늘린다.

**`--previous` 가 핵심이다.** 재시작된 파드의 현재 로그에는 죽기 직전 이야기가 없다.

## ArgoCD 로그인

### 초기 비밀번호

```sh
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

`-o jsonpath` 의 경로는 `-o yaml` 로 본 구조 그대로. Secret 값은 base64 라 디코딩.
끝의 `; echo` 는 비밀번호 뒤에 줄바꿈이 없어 프롬프트가 붙는 걸 막는다.

### 밖에서 접속

```sh
ssh -J uuook@<외부IP>:<포트> uuook@192.168.45.4 -L 30443:10.23.218.234:30443
```

`-J` 로 맥미니(공유기 포트포워딩)를 거쳐 우분투로, `-L` 은 **명령을 친 노트북**의 30443 을 cp1:30443 으로.
같은 세션에서 `multipass shell cp1` 로 들어가 비밀번호를 꺼냈다. `multipass exec cp1 -- ...` 로 한 줄에 하면
jsonpath 따옴표가 여러 겹이 돼서 헷갈린다.

### 로그인 후

UI 의 User Info 에서 비밀번호 변경. 바뀐 건 `argocd-secret` 의 `admin.passwordMtime` 으로 확인된다.
초기 Secret 은 이제 필요 없고, 남겨 두면 Secret 을 읽을 수 있는 누구나 admin 비밀번호를 알게 되므로 지웠다.

```sh
kubectl -n argocd delete secret argocd-initial-admin-secret
```

남은 Secret 중 `sh.helm.release.v1.argocd.v1` 은 **Helm 이 릴리스 기록을 저장하는 곳**이다. `helm list`·`helm history` 가
읽는 게 이것이고, `upgrade` 할 때마다 `v2`, `v3` 이 쌓인다. Helm 에는 별도 DB 가 없다.

## ArgoCD 가 git 을 읽을 키

ArgoCD 는 클러스터 안에서 우분투 bare 저장소(`uuook@192.168.45.4:git/homelab-k8s.git`)를 당겨 가야 한다.
**`uuook` 키를 그대로 주면 파드가 NAS 호스트에 셸 권한을 갖는다.** 두 층을 섞지 않는다는 원칙과 정면충돌.

### 두 안

| | 전용 키 + `authorized_keys` 제한 | `git-shell` 별도 계정 |
|---|---|---|
| 읽기 전용 | 됨 (`git-upload-pack` 만) | 안 됨, 파일 권한으로 따로 |
| 범위 | 저장소 하나 | 계정이 닿는 모든 저장소 |
| 바꿀 것 | 한 줄 | 계정·저장소 이동·push 주소 |

필요한 건 "저장소 하나 읽기"뿐이라 앞의 것. 실행 계정이 `uuook` 인 건 약점이지만 명령이 하나로 고정된다.
읽을 저장소가 늘면 저장소별 읽기 전용 deploy key 가 기본인 Gitea·Forgejo 로 간다.

### 만든 것

```sh
ssh-keygen -t ed25519 -N "" -C "argocd-readonly@homelab-k8s" -f ~/argocd-repo-key   # 저장소 밖에
```

```
restrict,from="10.23.218.0/24",command="git-upload-pack /home/uuook/git/homelab-k8s.git" ssh-ed25519 AAAA... argocd-readonly@homelab-k8s
```

| 옵션 | 뜻 |
|---|---|
| `restrict` | 포트포워딩·agent·X11·pty 전부 끔 |
| `from=` | VM 대역에서 온 접속만. 파드 트래픽은 노드(VM) IP 로 바뀌어 나온다 |
| `command=` | 클라이언트가 무슨 명령을 보내든 **이것만** 실행. push(`git-receive-pack`)는 불가능 |

### 한 번 틀렸다 — 옵션과 키가 두 줄로

```
4: restrict,from="10.23.218.0/24",command="git-upload-pack ..."
5: ssh-ed25519 AAAA... argocd-readonly@homelab-k8s
```

`authorized_keys` 는 **한 줄이 한 항목**이다. 4행은 키가 없어 조용히 무시되고, 5행은 옵션 없는 키라 그대로 통과.
결과: 이 키로 **셸이 열렸다.** 또 "틀린 설정이 에러를 안 낸다".

게다가 셸이 열린 걸 못 알아채고 그 안에서 push 시험을 이어서 쳐서, 그 시험은 cp1 이 아니라 우분투에서 돌았다
(에러의 `uuook@uuook-MS-7B89` 가 단서). 커밋 단계에서 신원 오류로 멈춰 push 는 없었다.
**시험 도중 결과가 예상과 다르면 거기서 멈출 것.** 셸이 열린 것 자체가 실패 신호였다.

vim 에서 4행 `J` 로 합쳐 고쳤다.

### 시험 — 아무것도 바꾸지 않고 확인하기

| 시험 | 어디서 | 결과 |
|---|---|---|
| `git ls-remote` | cp1 | `c996253 refs/heads/main` — 읽기 됨 |
| `ssh ... whoami` | cp1 | `whoami` 대신 git 프로토콜 문자열 — 셸 막힘 |
| `ssh ... "git-receive-pack <repo>"` | cp1 | 이것도 upload-pack 응답 — push 불가 |
| 같은 키로 접속 | 우분투 자신 | `Permission denied` — `from` 동작 |

push 를 실제로 해 보지 않고도 막혔는지 알 수 있다. 응답 첫 줄의 기능 목록(`multi_ack thin-pack ...`)은
upload-pack(읽기) 것이다. receive-pack(쓰기)이었다면 `report-status`, `delete-refs` 가 보였을 것.

### 다음

ArgoCD 에 줄 것 세 가지: 저장소 URL, 개인키, **호스트 키(known_hosts)**. 마지막을 빠뜨리면 `knownhosts: key is unknown`.
UI(Settings → Repositories)로 등록해 보고, 그 결과로 생긴 Secret 을 kubectl 로 들여다볼 예정.

