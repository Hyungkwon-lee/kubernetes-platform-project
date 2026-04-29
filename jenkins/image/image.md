# jenkins/image

기본 `jenkins/jenkins:lts` 이미지에는 `docker` CLI와 `kubectl`이 없어
파이프라인 내 빌드·배포 명령 실행이 불가합니다.
이를 해결하기 위해 필요한 CLI 툴만 최소 설치한 커스텀 이미지를 직접 빌드했습니다.

---

## 파일 구성

```
image
├── Dockerfile
└── install_tools.sh
```

---

## Dockerfile

```dockerfile
FROM jenkins/jenkins:lts

USER root
COPY install_tools.sh /usr/local/bin/install_tools.sh
RUN chmod +x /usr/local/bin/install_tools.sh && \
    /bin/bash /usr/local/bin/install_tools.sh && \
    command -v docker && \
    docker --version && \
    command -v kubectl && \
    kubectl version --client
USER jenkins
```

**설계 포인트**
- `USER root` → 툴 설치 → `USER jenkins` 복귀: 최소 권한 원칙 적용
- `command -v` + `--version`: 빌드 시점에 툴 설치 여부를 검증해 잘못된 이미지가 배포되는 것을 방지
- 설치 로직을 `install_tools.sh`로 분리: Dockerfile을 간결하게 유지하고 스크립트 재사용 가능

---

## install_tools.sh

### docker-ce-cli 설치

```bash
# Docker 공식 GPG 키 등록
curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc

# Docker apt 레포지토리 추가
echo "deb [arch=...] https://download.docker.com/linux/debian ... stable" \
  > /etc/apt/sources.list.d/docker.list

# CLI만 설치 (docker 데몬은 불필요)
apt-get install -y --no-install-recommends docker-ce-cli
```

`docker-ce` (데몬)가 아닌 `docker-ce-cli`만 설치합니다.
실제 빌드는 호스트 Docker 데몬(docker.sock)이 처리하므로 CLI만 있으면 충분합니다.

### kubectl 설치

```bash
KUBECTL_VERSION="v1.28.15"
curl -LO "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl"
install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm -f kubectl
```

클러스터 버전(`v1.28.15`)과 맞춰 고정 버전으로 설치합니다.
`curl | bash` 방식 대신 공식 바이너리를 직접 다운로드해 설치합니다.

### 이미지 경량화

```bash
apt-get clean
rm -rf /var/lib/apt/lists/*
```

apt 캐시를 제거해 이미지 레이어 크기를 줄입니다.
`--no-install-recommends` 옵션도 같은 목적으로 사용했습니다.

---

## 빌드 및 푸시

```bash
docker build -t hklee2748/jenkins:k8s-1.28.15-docker-v3 .
docker push hklee2748/jenkins:k8s-1.28.15-docker-v3
```

태그 네이밍: `k8s-{kubectl버전}-docker-v{빌드버전}` 형식으로 어떤 환경용 이미지인지 식별 가능하게 했습니다.

---

## 이슈: Docker socket 권한 문제 (DooD)

**문제**
Jenkins Pod에서 docker.sock을 마운트해 사용하는 구조에서
호스트와 컨테이너 간 GID 불일치로 `permission denied` 발생.

**시도한 해결책**
빌드 시점에 `HOST_GID`를 ARG로 받아 그룹 권한을 동적으로 설정하는 방식 시도 → 실패

**임시 해결**
`deployment.yaml`에서 `runAsUser: 0`으로 root 실행

**개선 방향**

| 방법 | 설명 | 장점 |
|------|------|------|
| Kaniko | 데몬 없이 컨테이너 내부에서 이미지 빌드 | root 불필요, 보안 향상 |
| Buildah | OCI 이미지 빌드 도구 | 데몬리스, rootless 지원 |
| Docker-in-Docker (DinD) | 컨테이너 내부에 Docker 데몬 실행 | sock 마운트 불필요 (단, 보안 이슈 존재) |
