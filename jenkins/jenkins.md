# jenkins/deploy

Jenkins Pod를 쿠버네티스에 배포하기 위한 매니페스트 구성입니다.
WAS 배포 구성과 대부분 유사하지만, Docker 빌드 실행을 위한 추가 설정이 필요합니다.

---

## 배포 구조

```
namespace → pv → pvc → deployment → service → ingress
```

## Jenkins 커스텀 이미지
이 배포에서 사용하는 이미지는 기본 Jenkins 이미지를 그대로 사용하지 않고 직접 빌드한 커스텀 이미지입니다.  
→ [커스텀 이미지 상세 보기](../image/image.md)

---

## 파일별 설명

### namespace.yaml

```yaml
name: user3-jenkins
labels:
  owner: user3
  app: jenkins
```

WAS와 동일하게 사용자별 네임스페이스를 분리했습니다.
`user3-jenkins` / `user3-was` 네임스페이스를 각각 구성해 리소스를 격리합니다.

---

### pv.yaml / pvc.yaml

```yaml
nfs:
  server: 192.168.20.20
  path: /mnt/nfs_share/jenkins/user3
```

Jenkins 홈 디렉토리(`/var/jenkins_home`)를 NFS에 마운트합니다.
플러그인, 파이프라인 설정, 빌드 히스토리 등이 저장되므로
Pod가 재생성되어도 데이터가 유지됩니다.

**Retain 정책**: PVC 삭제 시에도 NFS 서버의 데이터를 보존합니다.
Jenkins 설정을 처음부터 다시 잡는 비용이 크기 때문에 특히 중요합니다.

---

### deployment.yaml

WAS deployment와의 주요 차이점입니다.

**① runAsUser: 0 (root 실행)**
```yaml
securityContext:
  runAsUser: 0
```
docker.sock 권한 문제로 인한 임시 해결책입니다.
보안상 적절하지 않으며, 개선 방향은 `image.md` 참고.

**② docker.sock 마운트**
```yaml
volumes:
  - name: docker-sock
    hostPath:
      path: /var/run/docker.sock
      type: Socket
```
Jenkins Pod 내부에서 `docker build/push` 명령을 실행하기 위해
호스트의 Docker 데몬 소켓을 컨테이너에 마운트합니다. (DooD 방식)

**③ 포트 2개**
```yaml
ports:
  - containerPort: 8080   # Jenkins UI
  - containerPort: 50000  # Jenkins Agent 통신용
```
50000 포트는 Jenkins Agent(슬레이브)와의 통신에 사용됩니다.
현재는 Agent를 별도로 구성하지 않았지만 확장을 고려해 열어뒀습니다.

---

### service.yaml

```yaml
ports:
  - name: http
    port: 8080
    targetPort: 8080
  - name: agent
    port: 50000
    targetPort: 50000
```

WAS service가 `80 → 8080` 매핑인 것과 달리,
Jenkins는 `8080 → 8080` 그대로 사용합니다.

---

### ingress.yaml

```yaml
host: jenkins.user3.goping.com
backend:
  port: 8080
```

WAS ingress(`was.user3.goping.com`)와 함께 동일한 Ingress Controller에서
host 기반으로 분기됩니다.

---

## 적용 명령어

```bash
kubectl apply -f namespace.yaml
kubectl apply -f pv.yaml
kubectl apply -f pvc.yaml
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f ingress.yaml
```

---

## 개선 가능한 부분

- `runAsUser: 0` 제거 → Kaniko 또는 Buildah로 전환해 root 없이 빌드
- Jenkins Agent를 별도 Pod로 분리하면 빌드 부하를 Master와 분산 가능
- 50000 포트에 대한 별도 Service/Ingress 구성으로 외부 Agent 연결 가능
