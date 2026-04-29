# app/deploy

Spring-PetClinic WAS를 쿠버네티스에 배포하기 위한 매니페스트 구성입니다.

---

## 배포 구조

namespace → pv → pvc → deployment → service → ingress

적용 순서대로 의존관계가 있습니다. namespace가 없으면 나머지 리소스를 생성할 수 없고,
pv가 없으면 pvc가 바인딩되지 않습니다.

---

## 파일별 설명

### namespace.yaml

```yaml
name: user3-was
labels:
  owner: user3
  app: was
```

멀티테넌트 구조에서 사용자별 리소스를 격리하기 위해 별도 네임스페이스를 구성했습니다.
label로 소유자와 앱을 명시해 리소스 식별을 용이하게 했습니다.

---

### pv.yaml

```yaml
capacity: 10Gi
accessModes: ReadWriteMany
storageClassName: nfs
persistentVolumeReclaimPolicy: Retain
nfs:
  server: 192.168.20.20
  path: /mnt/nfs_share/was/user3
```

**ReadWriteMany**: replicas가 2개이므로 여러 Pod에서 동시에 마운트 가능한 RWX 모드 필요.  
NFS는 RWX를 지원하므로 적합합니다.

**Retain 정책**: PVC 삭제 시에도 NFS 서버의 데이터를 보존합니다.  
Delete 정책은 PVC 삭제 시 데이터도 함께 삭제되므로 운영 환경에서는 위험합니다.

> NFS 서버: `192.168.20.20` (서버 VLAN 내 별도 구성)

---

### pvc.yaml

```yaml
accessModes: ReadWriteMany
storage: 10Gi
storageClassName: nfs
volumeName: pv-user3-was
```

`volumeName`을 명시해 특정 PV와 1:1로 바인딩했습니다.  
명시하지 않으면 조건에 맞는 PV가 자동으로 선택되는데, 멀티테넌트 환경에서는
다른 사용자의 PV에 바인딩될 수 있어 명시적으로 지정했습니다.

---

### deployment.yaml

```yaml
replicas: 2
image: hklee2748/spring-petclinic:latest
imagePullPolicy: Always
```

**replicas: 2**: 기본 2개로 구성. Jenkins 파이프라인 배포 후 `kubectl scale`로 조정 가능합니다.

**imagePullPolicy: Always**: 태그가 `latest`이므로 Pod 재시작 시 항상 최신 이미지를 pull합니다.  
`IfNotPresent`로 설정하면 같은 태그라도 로컬 캐시를 사용해 업데이트가 반영되지 않을 수 있습니다.

> Jenkins 파이프라인에서는 `BUILD_NUMBER` 태그를 사용하지만,  
> 수동 배포 시 편의를 위해 `latest`도 병행해서 push하는 방식을 사용했습니다.

---

### service.yaml

```yaml
type: ClusterIP
port: 80
targetPort: 8080
```

**ClusterIP**: 외부 직접 노출 없이 Ingress를 통해서만 접근하는 구조이므로 ClusterIP로 충분합니다.

**포트 매핑**: Spring-PetClinic은 8080으로 뜨지만, 서비스는 80으로 노출해
Ingress에서 표준 HTTP 포트로 라우팅합니다.

---

### ingress.yaml

```yaml
ingressClassName: nginx
host: was.user3.goping.com
path: /
backend:
  service: user3-was-svc
  port: 80
```

Host 기반 라우팅으로 `was.user3.goping.com` 요청을 `user3-was-svc`로 전달합니다.  
동일 Ingress Controller에서 `jenkins.user3.goping.com`과 host로 분기됩니다.

> DNS: `192.168.20.11` 에 도메인 등록 후 MetalLB가 할당한 IP로 연결

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

- `latest` 태그 대신 항상 `BUILD_NUMBER` 태그만 사용하는 방식으로 통일하면 버전 추적이 명확해집니다.
- Ingress에 TLS 설정을 추가하면 HTTPS 접근이 가능합니다. (cert-manager 연동)
- HPA(HorizontalPodAutoscaler)를 추가하면 트래픽에 따라 replicas를 자동 조정할 수 있습니다.