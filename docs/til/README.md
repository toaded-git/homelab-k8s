# kubeadm 3노드 클러스터 구축 TIL

집 서버에 쿠버네티스를 올려보며 배운 것들. 회사에서는 k8s 를 쓰지 않아 순수 학습 목적이고,
**설치·설정·명령 실행은 전부 직접** 했다. 막힌 곳과 왜 그랬는지를 같이 남긴다.

| | |
|---|---|
| 기간 | 2026-09-14 ~ 09-29 (실작업 5일) |
| 호스트 | Ubuntu 24.04, Ryzen 5 3600 / 32GB / GTX 1660 Super |
| 노드 | multipass VM 3대 (`cp1`·`w1`·`w2`), 각 2코어 4GB 20GB, Ubuntu 26.04, NAT 10.23.218.0/24 |
| 스택 | kubeadm·kubelet·kubectl v1.37.0, containerd 2.2.2, Calico v3.32.2 |
| 목표 | 빈 VM → 3노드 클러스터 → **내가 만든 앱**을 사설 레지스트리 경유로 배포 |

같은 호스트에서 NAS(Docker Compose: Jellyfin·Immich·Samba·restic·Uptime Kuma)가 돌고 있어서,
**NAS 를 멈추지 않는 것**이 모든 판단의 제약이었다.

## 목차

| 날 | 날짜 | 제목 | 핵심 |
|---|---|---|---|
| 1 | 09-14 | [설치하기 전에 멈춰 선 날](day1-2026-09-14-plan.md) | 호스트에 직접 못 올리는 이유, VM 으로 분리 결정 |
| 2 | 09-17 | [커널부터 맞추기](day2-2026-09-17-node-prereq.md) | `br_netfilter`·`ip_forward` 가 왜 필요한가 |
| 3 | 09-18 | [클러스터가 서다](day3-2026-09-18-cluster-up.md) | cgroup 드라이버, static pod, **Calico IP 풀이 집 LAN 과 충돌** |
| 4 | 09-28 | [Service 를 믿지 말고 확인하기](day4-2026-09-28-service-and-image.md) | Service 는 확률 분기 iptables 규칙, 재부팅 생존 |
| 5 | 09-29 | [사설 레지스트리와 첫 배포](day5-2026-09-29-registry-and-deploy.md) | containerd 2.x `certs.d`, CrashLoop 진단, ConfigMap |

## 지금 상태

```
[맥] git push ──> [우분투] bare repo ──> clone ──> docker build
                                                      │
                                              docker push (평문 HTTP 예외)
                                                      ▼
                          ┌─── VM 3대 (10.23.218.0/24) ────────────┐
                          │  registry:2  NodePort 30500 (emptyDir) │
                          │        ▲ pull (certs.d)                │
                          │  dashboard x2  NodePort 30080          │
                          │    /config ← ConfigMap                 │
                          └────────────────────────────────────────┘
```

## 남은 것

- Secret 으로 SSH 키 주입 (`defaultMode: 0400`)
- PVC — 레플리카 2 + RWO 는 노드가 갈리면 한쪽이 못 붙는다. 이 충돌을 어떻게 풀지가 다음 과제
- readiness / liveness probe
- 인그레스 컨트롤러 직접 설치, Calico NetworkPolicy
- 그다음 호스트에 k3s, 최종 목표는 **git push 하면 배포되는 개인 PaaS**

## 돌아보며

쿠버네티스 설치 자체는 하루면 된다. 시간이 걸린 건 전부 **기본값이 내 환경과 충돌하는 지점**이었다.
Calico 의 기본 IP 풀이 집 공유기 대역과 겹치고, 문서 예제가 containerd 1.x 기준이고,
`daemon.json` 에 이미 남이 쓰던 설정이 들어 있었다.

그리고 Day 3 까지는 "붙이기"였는데 Day 4 부터는 계속 **디버깅**이었다.
실무 k8s 시간의 대부분이 여기라는 걸 알게 됐다. 설치는 한 번이고 디버깅은 매일이니까.
