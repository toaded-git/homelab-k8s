# platform

kubeadm 실습 클러스터(multipass VM `cp1`·`w1`·`w2`)에 올리는 매니페스트.
NAS 층(Docker Compose)과 섞지 않는다.

지금은 cp1 `/home/ubuntu/` 에서 직접 편집하고 `kubectl apply` 하는 중이라,
여기 있는 파일은 **그 사본**이다. cp1 에서 고쳤으면 여기로 다시 가져올 것.

```sh
# 맥의 저장소 루트에서. 따옴표 밖의 > 는 맥 쪽 셸이 처리한다
ssh uuook@192.168.45.4 'multipass exec cp1 -- cat /home/ubuntu/dashboard-deploy.yaml' </dev/null \
  > platform/dashboard/dashboard-deploy.yaml
```

## registry/

클러스터 안 이미지 레지스트리 (`registry:2`, NodePort 30500).
저장소는 PVC `registry-data` (`/var/lib/registry`). 파드를 다시 만들어도 이미지가 남는다.
local-path 라 PV 가 처음 뜬 노드(**w1**) 디스크에 있고, 파드도 w1 에만 뜬다. w1 이 죽으면 레지스트리도 멈춘다.
NodePort 라 접속 주소는 어느 노드 IP 든 상관없다.

적용은 **PVC 먼저, Deployment 다음.** 반대로 하면 `persistentvolumeclaim not found` 로 잠깐 스케줄링이 실패한다(곧 재시도로 붙긴 한다).

```sh
curl http://10.23.218.234:30500/v2/_catalog                     # 저장소 목록
curl http://10.23.218.234:30500/v2/homelab-dashboard/tags/list  # 태그 목록
```

이 레지스트리를 쓰려면 양쪽에 평문 HTTP 예외가 필요하다 (이미 적용됨).

- 우분투 호스트 (push): `/etc/docker/daemon.json` 의 `insecure-registries`
- VM 3대 (pull): `/etc/containerd/certs.d/10.23.218.234:30500/hosts.toml`,
  `config.toml` 의 `[plugins.'io.containerd.cri.v1.images'.registry] config_path`

`10.23.218.234` 는 cp1 IP. NodePort 라 아무 노드 IP 를 써도 같은 파드로 간다.
multipass 가 IP 를 다시 주면 두 설정과 이미지 태그를 같이 고쳐야 한다.

## dashboard/

`homelab-dashboard` (Go+React, 8080). 이미지는 위 레지스트리에서 pull.

ConfigMap 과 Secret 은 **원본이 우분투 `~/homelab-dashboard/config/` 에 있어서 여기 없다.**

```sh
# 우분투 호스트에서 cp1 으로 옮긴 뒤, cp1 에서
kubectl create configmap dashboard-config --from-file=config.yaml --from-file=known_hosts
kubectl create secret generic dashboard-ssh --from-file=id_ed25519
```

- `config.yaml` 의 `key_path` 는 `/secrets/id_ed25519` 로 바꿔서 쓴다.
  ConfigMap 과 Secret 을 같은 `/config` 에 겹쳐 마운트할 수 없기 때문.
- Secret 은 base64 일 뿐 암호화가 아니다. **개인키가 든 YAML 은 커밋하지 않는다.**
