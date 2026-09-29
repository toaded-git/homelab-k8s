# Day 4 (2026-09-28) — Service 를 믿지 말고 확인하기 / 내 앱 준비

← [Day 3](day3-2026-09-18-cluster-up.md) · [목차](README.md) · [다음: Day 5](day5-2026-09-29-registry-and-deploy.md)

## 한 일

- Service 로드밸런싱을 **로그로 실증**: ClusterIP 에 30번 요청 → 파드 3개 access log 가 10 / 5 / 15 로 갈림
- 그 밑의 iptables 규칙까지 추적
- 9/27 호스트 재부팅 후 클러스터 상태 확인
- 앱 선정: `homelab-dashboard` (Go+React, 단일 컨테이너, 8080)
- 우분투에 bare 저장소 `~/git/homelab-dashboard.git` → 맥에서 push → 우분투에서 clone → `docker build` (23.5MB)
- 클러스터 안에 `registry:2` 를 NodePort 30500 으로 배포 (매니페스트 직접 작성)

## 막힌 곳

### 매니페스트를 처음부터 손으로 쓰려니 필드 구조가 안 떠올랐다

→ 뼈대를 뽑고 고치는 방식으로 전환.

```sh
kubectl create deployment web --image=nginx --replicas=3 --dry-run=client -o yaml > web.yaml
```

`--dry-run=client` 는 클러스터에 아무것도 보내지 않고 YAML 만 찍어준다.

### 호스트를 재부팅했는데 클러스터가 어떻게 됐는지 알 수 없었다

→ 확인해 보니 **스스로 복구**되어 있었다 (아래).

## 배운 것

### Service 는 프록시 서버가 아니라 iptables 규칙이다

파드 3개에 30번 요청을 보내니 **10 / 5 / 15** 로 갈렸다. 처음엔 고장인가 싶었는데, 규칙을 보면 정상이다.

```
statistic mode random probability 0.33333  -> 파드 A
statistic mode random probability 0.50000  -> 파드 B   (남은 것 중 1/2)
                                           -> 파드 C
```

**확률 분기**라서 균등하지 않다. 라운드로빈이 아니다.
별도 프로세스가 트래픽을 받아 넘기는 게 아니라, 커널이 목적지 주소를 바꿔치기(DNAT)한다.
그래서 Service 는 죽지도 않고 지연도 거의 없다.

### 재부팅 생존 — 파드 IP 를 참조하면 안 되는 이유

호스트 재부팅 후 **multipass VM 자동 시작 → containerd → kubelet → 파드 재생성** 순으로 알아서 복구됐다.

- 파드 IP 는 **전부 바뀌었다**
- ClusterIP 는 **그대로였고**, iptables 규칙이 새 파드 IP 로 자동 갱신됐다

"파드 IP 를 직접 참조하지 말라"는 말의 의미를 눈으로 봤다.
(단 VM IP 가 그대로였던 건 multipass DHCP 운이지 보장이 아니다. 바뀌면 API 주소가 깨진다.)

### `skopeo` vs `docker push`

- `skopeo` — 데몬 없이 레지스트리 간 이미지를 직접 복사한다. CI 에서 유용
- `docker push` — 로컬 Docker 데몬의 이미지 저장소를 거친다

빌드를 Docker 로 했으니 후자가 자연스럽다.

### NodePort 는 30000~32767

이 범위 밖을 쓰려면 apiserver 설정을 바꿔야 한다. 레지스트리 30500, 앱 30080 으로 나눠 잡았다.

### 사설 레지스트리의 닭-달걀

`registry:2` 이미지 자체는 Docker Hub 에서 받아온다. 클러스터 자기 레지스트리에 자기를 넣을 수는 없다.
그리고 저장소를 `emptyDir` 로 두면 **파드를 다시 만들 때 올려둔 이미지가 전부 사라진다** (다음 과제: PVC).

---
← [Day 3](day3-2026-09-18-cluster-up.md) · [목차](README.md) · [다음: Day 5](day5-2026-09-29-registry-and-deploy.md)
