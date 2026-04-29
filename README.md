# kubernetes-platform-project

Jenkins + Docker Hub + Kubernetes 기반 CI/CD 파이프라인 구축 프로젝트입니다.  
→ 프로젝트 전체 개요는 [infra-portfolio](https://github.com/Hyungkwon-lee/infra-portfolio) 참고

---

## CI/CD 흐름

GitHub (main) → Maven Build → Docker Build → Docker Hub Push → kubectl rolling update

---

## 디렉토리 구조 및 문서

| 경로 | 설명 | 문서 |
|------|------|------|
| `k8s/` | 클러스터 컴포넌트 (Calico, MetalLB, ingress-nginx) | [k8s.md](./k8s/k8s.md) |
| `jenkins/image/` | 커스텀 Jenkins 이미지 | [image.md](./jenkins/image/image.md) |
| `jenkins/deploy/` | Jenkins 매니페스트 | [jenkins.md](./jenkins/jenkins.md) |
| `app/deploy/` | WAS(Spring-PetClinic) 매니페스트 | [app.md](./app/app.md) |
| `Jenkinsfile` | CI/CD 파이프라인 전체 정의 | - |

---

## 담당 작업

- NFS 서버 구축 및 Worker Node 마운트 설정
- Jenkins 커스텀 Docker 이미지 설계 및 빌드
- Jenkins / WAS 전체 K8s 매니페스트 작성
- Jenkinsfile 작성 (5단계 파이프라인 전체)

---

## 기술 스택

`Kubernetes` `Jenkins` `Docker` `Docker Hub` `Maven` `Spring-PetClinic`  
`MetalLB` `ingress-nginx` `NFS` `Calico`
