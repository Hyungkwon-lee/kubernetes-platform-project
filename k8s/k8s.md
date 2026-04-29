# k8s 클러스터 컴포넌트

쿠버네티스 클러스터 운영에 필요한 네트워크 및 로드밸런싱 컴포넌트 구성입니다.
공식 매니페스트를 기반으로 설치하고, 환경에 맞게 설정을 추가했습니다.

---

## 구성 요소

### Calico (CNI)

**역할**: Pod 간 네트워크 통신 및 네트워크 정책 관리

**선택 이유**  
쿠버네티스는 CNI 플러그인이 없으면 Pod 간 통신이 불가합니다.  
Calico는 가장 널리 사용되는 CNI 중 하나로, NetworkPolicy 지원과 레퍼런스가 풍부해 선택했습니다.

**대안 비교**
| 플러그인 | 특징 |
|----------|------|
| Calico | NetworkPolicy 지원, 레퍼런스 풍부 |
| Flannel | 설정 단순, NetworkPolicy 미지원 |
| Cilium | eBPF 기반, 고성능이나 학습 난이도 높음 |

```bash
kubectl apply -f calico/calico.yaml
```

---

### ingress-nginx (Ingress Controller)

**역할**: 외부 HTTP 요청을 클러스터 내부 서비스로 라우팅

**선택 이유**  
MetalLB로 외부 IP를 할당받은 후, Host 기반 라우팅으로 서비스를 분기하기 위해 사용했습니다.  
ingress-nginx는 공식 지원이 안정적이고 레퍼런스가 많아 선택했습니다.

**대안 비교**
| 컨트롤러 | 특징 |
|----------|------|
| ingress-nginx | 공식 지원, 레퍼런스 풍부 |
| Traefik | 자동 설정, 대시보드 제공 |
| HAProxy Ingress | 고성능, 설정 복잡 |

```bash
kubectl apply -f ingress-nginx/deploy.yaml
```

---

### MetalLB (Load Balancer)

**역할**: 온프레미스 환경에서 `LoadBalancer` 타입 서비스에 외부 IP 할당

**선택 이유**  
클라우드 환경과 달리 온프레미스에서는 `LoadBalancer` 타입 서비스를 생성해도  
외부 IP가 자동으로 할당되지 않습니다. MetalLB는 이를 해결하기 위한 솔루션입니다.

**동작 방식**: L2 모드 (ARP 기반)  
BGP 모드는 라우터 설정이 필요해 학습 환경에서 오버헤드가 크다고 판단, L2 모드를 선택했습니다.

```bash
kubectl apply -f metallb/metallb-native.yaml
kubectl apply -f metallb/metallb-config.yaml
```

**metallb-config.yaml 설명**

```yaml
addresses:
  - 192.168.20.150-192.168.20.199  # 클러스터가 속한 서버 VLAN 대역에서 할당
autoAssign: true                    # LoadBalancer 서비스 생성 시 자동 IP 할당
```

> IP 대역은 서버 VLAN(192.168.20.0/24) 내에서 다른 장비와 충돌하지 않는 범위로 설정했습니다.

---

## 설치 순서

```bash
# 1. CNI 먼저 설치 (없으면 Pod가 NotReady 상태)
kubectl apply -f calico/calico.yaml

# 2. MetalLB 설치 후 IP 풀 설정
kubectl apply -f metallb/metallb-native.yaml
kubectl apply -f metallb/metallb-config.yaml

# 3. Ingress Controller 설치
kubectl apply -f ingress-nginx/deploy.yaml
```

> CNI → MetalLB → ingress-nginx 순서로 설치해야 합니다.  
> CNI가 없으면 이후 Pod들이 정상적으로 뜨지 않습니다.