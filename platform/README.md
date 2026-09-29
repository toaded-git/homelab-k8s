# platform

kubeadm 실습 클러스터(multipass VM `cp1`·`w1`·`w2`)에 올리는 매니페스트.
NAS 층(Docker Compose)과 섞지 않는다.

지금은 cp1 `/home/ubuntu/` 에서 직접 편집하고 `kubectl apply` 하는 중이라,
여기 있는 파일은 **그 사본**이다. cp1 에서 고쳤으면 여기로 다시 가져올 것.

```sh
multipass exec cp1 -- cat /home/ubuntu/dashboard-deploy.yaml > platform/dashboard/dashboard-deploy.yaml
```

## registry/

클러스터 안 이미지 레지스트리 (`registry:2`, NodePort 30500).
저장소가 `emptyDir` 이라 파드를 다시 만들면 올려둔 이미지가 사라진다. 그땐 우분투에서 다시 push.

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
