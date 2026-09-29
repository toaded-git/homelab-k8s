# Day 2 (2026-09-17) — 커널부터 맞추기

← [Day 1](day1-2026-09-14-plan.md) · [목차](README.md) · [다음: Day 3](day3-2026-09-18-cluster-up.md)

## 한 일

- multipass 로 VM 3대 생성 (`cp1`·`w1`·`w2`, 각 2코어 4GB 20GB, Ubuntu 26.04)
- 커널 모듈 `overlay`·`br_netfilter` 적재 + `/etc/modules-load.d/k8s.conf` 로 부팅 시 자동 적재
- sysctl 설정 + `/etc/sysctl.d/k8s.conf`
  ```
  net.bridge.bridge-nf-call-iptables  = 1
  net.bridge.bridge-nf-call-ip6tables = 1
  net.ipv4.ip_forward                 = 1
  ```
- swap 없음 확인
- **재부팅 후에도 유지되는지 확인** (이게 핵심)

## 막힌 곳

### `/etc/modules-load.d/` 디렉터리가 세 대 모두 없었다

문서는 "이 파일을 만들어라"라고만 해서 디렉터리가 당연히 있다고 가정했는데, 최소 설치 이미지에는 없었다.
직접 만들고 재부팅으로 검증했다. `/etc/sysctl.d/` 는 있었다.

### multipass 데몬이 반복적으로 물렸다

`restart`·`force stop`·`delete` 에서 멈췄다. VM 자체는 멀쩡한데 상태가 `Starting` 에서 안 넘어온다.

→ `sudo snap restart multipass` 로 해소. 단 **qemu 가 데몬의 자식 프로세스**라 데몬을 재시작하면 VM 도 같이 꺼진다.

## 배운 것

### 왜 `br_netfilter` 인가

파드는 노드 안의 리눅스 브리지에 붙는다. 기본적으로 **브리지를 지나는 트래픽은 iptables 를 타지 않는다.**
그런데 Service 는 kube-proxy 가 만든 iptables 규칙으로 구현된다.

이 모듈을 켜고 `bridge-nf-call-iptables = 1` 로 해야 브리지 트래픽이 iptables 를 거쳐서 **Service 가 동작한다.**
빠뜨리면 노드는 Ready 인데 Service 로 접속이 안 되는, 원인 찾기 어려운 상태가 된다.

### 왜 `ip_forward` 인가

노드가 파드 간 패킷을 중계하는 **라우터 노릇**을 한다. 꺼져 있으면 자기 앞으로 오지 않은 패킷을 그냥 버린다.

### 커널 모듈이란

커널의 플러그인이다. 필요한 기능만 실행 중에 끼웠다 뺄 수 있다 (`lsmod`·`modprobe`).

- `modprobe overlay` — **지금** 켜는 것
- `/etc/modules-load.d/k8s.conf` — **다음 부팅에도** 켜지게 하는 것

**둘 다 해야 재부팅을 견딘다.** 한쪽만 하면 "어제는 됐는데 오늘 안 된다"가 된다. sysctl 도 같은 구조다
(`sysctl -w` 는 지금, `/etc/sysctl.d/` 는 영구).

그래서 이 단계의 검증은 명령이 성공하는 게 아니라 **재부팅 후에도 값이 살아 있는 것**이다.

---
← [Day 1](day1-2026-09-14-plan.md) · [목차](README.md) · [다음: Day 3](day3-2026-09-18-cluster-up.md)
