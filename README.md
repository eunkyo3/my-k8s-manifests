# ArgoCD & Kubernetes 인프라 구축 및 배포 실습 가이드

본 가이드는 쿠버네티스 클러스터 구축부터 ArgoCD를 이용한 GitOps 파이프라인 검증까지의 전 과정을 다룹니다.

## 1단계: OS 최적화 및 쿠버네티스(K3s) 설치

가상 머신 환경에서 자원 효율성을 극대화하기 위해 경량화된 K3s를 사용합니다.

1. **시스템 패키지 업데이트**
```bash
# 패키지 리스트 갱신 및 설치된 패키지 업그레이드
sudo apt update && sudo apt upgrade -y

```


2. **K3s 설치**
```bash
# K3s 설치 스크립트 실행
curl -sfL https://get.k3s.io | sh -

```


3. **권한 설정 및 클러스터 확인**
```bash
# 설정 파일 권한 변경 (kubectl 사용 권한 부여)
sudo chmod 644 /etc/rancher/k3s/k3s.yaml

# 노드 상태가 'Ready'인지 확인
kubectl get nodes

```



## 2단계: ArgoCD 배포 및 네트워크 설정

쿠버네티스 클러스터 위에 ArgoCD 엔진을 설치하고 외부에서 접속 가능하도록 설정합니다.

1. **ArgoCD 전용 네임스페이스 생성**
```bash
kubectl create namespace argocd

```


2. **ArgoCD 설치**
```bash
# 공식 매니페스트를 통한 리소스 배포
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

```


3. **서비스 노출 (NodePort 설정)**
```bash
# 외부 접속이 가능하도록 서비스 타입을 NodePort로 변경
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'

```


4. **접속 포트 확인**
```bash
# 서비스 목록에서 80:XXXXX/TCP 형태의 5자리 포트 번호 확인
kubectl get svc -n argocd argocd-server

```



## 3단계: 관리자 인증 및 UI 로그인

보안을 위해 초기 생성된 비밀번호를 확인하고 로그인합니다.

1. **초기 비밀번호 확인**
```bash
# 시크릿 객체에서 비밀번호 추출 및 디코딩
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

```


2. **UI 접속**
* 웹 브라우저에서 `https://<Ubuntu_IP>:<확인한_포트>` 주소로 접속합니다.
* 사용자 이름: `admin`
* 비밀번호: 위에서 확인한 문자열 입력



## 4단계: GitOps 애플리케이션 등록 (ArgoCD UI)

GitHub 저장소와 클러스터 상태를 동기화하기 위한 애플리케이션을 생성합니다.

1. **NEW APP 설정**
* **Application Name**: `my-gitops-app`
* **Project Name**: `default`
* **Sync Policy**: `Automatic` 선택 (Prune Resources, Self-heal 체크)


2. **Source & Destination 설정**
* **Repository URL**: 본인의 GitHub 저장소 주소 (예: `https://github.com/eunkyo3/my-k8s-manifests`)
* **Path**: `.` (YAML 파일 위치)
* **Cluster URL**: `https://kubernetes.default.svc`
* **Namespace**: `default`


3. **생성 완료**
* **CREATE** 버튼 클릭 후 애플리케이션 카드가 **Healthy** 및 **Synced** 상태가 되는지 확인합니다.



## 5단계: 시스템 안정성 및 동기화 검증 테스트

### 5.1 자가 치유(Self-healing) 테스트

시스템 장애 상황을 가정하여 수동으로 리소스를 삭제했을 때 복구되는지 확인합니다.

1. **Pod 삭제**: 터미널에서 `kubectl delete pod <Pod_이름>` 명령어로 실행 중인 Pod를 강제 삭제합니다.
2. **복구 확인**: ArgoCD UI에서 삭제된 Pod가 즉시 제거되고, 설정된 상태(Replicas)를 유지하기 위해 새로운 Pod가 자동으로 생성되는지 관찰합니다.

### 5.2 YAML 변경 및 자동 동기화 테스트

Git 저장소의 설정 변경이 실제 인프라에 실시간으로 반영되는지 확인합니다.

1. **GitHub 수정**: GitHub 저장소의 `deployment.yaml` 파일에서 `replicas` 값을 수정(예: 2 -> 3)하거나 컨테이너 이미지를 변경하고 **Commit & Push** 합니다.
2. **배포 확인**: ArgoCD가 수 초 내에 Git의 변경 사항을 감지하여 클러스터의 Pod 개수를 자동으로 늘리거나 이미지를 교체하는지 확인합니다.
3. **일관성 검증**: 터미널에서 `kubectl get pods` 명령어를 입력하여 변경된 설정대로 리소스가 운용되고 있는지 최종 확인합니다.

---

**출처:** [ArgoCD Getting Started](https://argo-cd.readthedocs.io/en/stable/getting_started/), [K3s Documentation](https://docs.k3s.io/)
