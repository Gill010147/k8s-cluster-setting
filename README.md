# Kubernetes 클러스터 구축 가이드 (Ubuntu 24.04 LTS)

[![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.30-blue.svg)](https://kubernetes.io/)
[![Ubuntu](https://img.shields.io/badge/Ubuntu-24.04%20LTS-orange.svg)](https://ubuntu.com/)
[![CNI](https://img.shields.io/badge/CNI-Calico-green.svg)](https://projectcalico.org/)
[![Container Runtime](https://img.shields.io/badge/Runtime-containerd-yellow.svg)](https://containerd.io/)

본 가이드는 VMware 환경에서 Ubuntu 24.04 LTS를 사용하여 마스터 노드 1개와 워커 노드 2개로 구성된 Kubernetes 클러스터를 처음부터 구축하는 과정을 상세히 다룹니다. 실제 운영 환경에서 활용할 수 있는 베스트 프랙티스와 트러블슈팅 가이드를 포함하여 안정적인 클러스터 운영을 위한 종합적인 정보를 제공합니다.

## 📋 목차

- [인프라 아키텍처](#-인프라-아키텍처)
- [사전 준비사항](#-사전-준비사항)
- [Kubernetes 환경 구성](#-kubernetes-환경-구성)
- [마스터 노드 설정](#-마스터-노드-설정)
- [워커 노드 설정](#-워커-노드-설정)
- [클러스터 검증](#-클러스터-검증)
- [네트워크 설정 및 서비스 타입](#-네트워크-설정-및-서비스-타입)
- [트러블슈팅 가이드](#-트러블슈팅-가이드)
- [운영 모범 사례](#-운영-모범-사례)

## 🏗️ 인프라 아키텍처

### 클러스터 구성

| 노드명 | 역할 | IP 주소 | 사양 |
|--------|------|---------|------|
| myserver01 | Master Node | 10.0.2.15 | 2 CPU, 4GB RAM |
| myserver02 | Worker Node | 10.0.2.20 | 2 CPU, 2GB RAM |
| myserver03 | Worker Node | 10.0.2.25 | 2 CPU, 2GB RAM |

<details>
  <summary>사진</summary>
  <img width="767" height="518" alt="image" src="https://github.com/user-attachments/assets/d8e8256b-a9c6-460a-9465-cfe17279c997" />
  <img width="792" height="470" alt="image" src="https://github.com/user-attachments/assets/28c91487-b1ae-4aef-aa40-0a44f3d93ac9" />


  - ova파일 가져오기 또는 복제로 세 개의 VM 설치
</details>


### 기술 스택

- **OS**: Ubuntu 24.04.2 LTS
- **컨테이너 런타임**: containerd
- **Kubernetes**: v1.30
- **CNI**: Calico
- **네트워크**: 192.168.0.0/16 (Pod Network CIDR)

### 클러스터 아키텍처 다이어그램
<img width="947" height="431" alt="image" src="https://github.com/user-attachments/assets/729a5039-8790-48d2-903e-ca7647969ee0" />


## 🚀 사전 준비사항

### VM 환경 설정

모든 노드에서 동일하게 수행해야 하는 기본 설정입니다.

#### 1. 호스트명 설정

각 노드별로 고유한 호스트명을 설정합니다.

```bash
# 각 노드에서 해당하는 호스트명으로 변경
sudo hostnamectl set-hostname myserver01  # 마스터 노드
sudo hostnamectl set-hostname myserver02  # 워커 노드 1
sudo hostnamectl set-hostname myserver03  # 워커 노드 2

# 설정 확인
hostname
sudo reboot  # 재부팅으로 설정 적용
```

#### 2. 네트워크 설정

정적 IP를 설정하여 클러스터 간 안정적인 통신을 보장합니다.

```bash
# 기존 네트워크 설정 파일 삭제
sudo rm /etc/netplan/50-cloud-init.yaml

# 새로운 네트워크 설정 파일 생성
sudo vi /etc/netplan/01-netcfg.yaml
```

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      addresses:
        - 10.0.2.15/24  # 각 노드별로 IP 변경 (15, 20, 25)
      routes:
        - to: default
          via: 10.0.2.1
      nameservers:
        addresses:
          - 8.8.8.8
      dhcp4: false
```

```bash
# 네트워크 설정 적용
sudo netplan apply
ip a  # IP 설정 확인
```


<details> 
  <summary> 네트워크 설정 확인 스크린샷 </summary>
  <img width="1104" height="302" alt="image" src="https://github.com/user-attachments/assets/a1c77981-6503-40a2-b272-93e9b413f6cb" />

  <img width="1015" height="255" alt="image" src="https://github.com/user-attachments/assets/ff7e4b84-94bd-483a-803d-42446d71dda6" />
  <img width="1017" height="260" alt="image" src="https://github.com/user-attachments/assets/a5ce7b24-f864-4d0e-b586-93ab93a28885" />

</details>


#### 3. SSH 키 기반 인증 설정

클러스터 노드 간 패스워드 없는 통신을 위한 SSH 키 설정입니다.

```bash
# RSA 키 페어 생성 (4096비트 보안 강화)
ssh-keygen -t rsa -b 4096

# 각 노드에서 다른 노드들에 공개키 복사
# 마스터 노드(myserver01)에서 실행
ssh-copy-id ubuntu@myserver02
ssh-copy-id ubuntu@myserver03

# 워커 노드들에서도 동일하게 설정
ssh-copy-id ubuntu@myserver01
ssh-copy-id ubuntu@myserver03  # myserver02에서
ssh-copy-id ubuntu@myserver01
ssh-copy-id ubuntu@myserver02  # myserver03에서

# 연결 테스트
ssh myserver02  # 패스워드 없이 접속 확인
```

## ⚙️ Kubernetes 환경 구성

### 커널 모듈 및 네트워크 설정

Kubernetes가 요구하는 시스템 수준의 네트워크 설정을 구성합니다.

```bash
# 필요한 커널 모듈 로드 설정
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

# 커널 모듈 즉시 로드
sudo modprobe overlay
sudo modprobe br_netfilter

# 네트워크 설정 파라미터
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

# 설정 적용
sudo sysctl --system
```

<details> <summary>커널 모듈 로드 확인 스크린샷</summary> 
  <img width="550" height="720" alt="image" src="https://github.com/user-attachments/assets/7b81d010-040b-4d8d-8c3c-fd0ced924c4e" />
</details>

### 컨테이너 런타임 설치

containerd를 Kubernetes의 컨테이너 런타임으로 설정합니다.

```bash
# containerd 설치
sudo apt update && sudo apt install -y containerd

# 버전 확인
containerd --version

# containerd 설정 디렉토리 생성
sudo mkdir -p /etc/containerd

# 기본 설정 파일 생성
sudo containerd config default | sudo tee /etc/containerd/config.toml > /dev/null

# SystemdCgroup을 true로 설정 (Kubernetes 호환성)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

# containerd 서비스 재시작
sudo systemctl restart containerd
sudo systemctl status containerd  # 상태 확인
```

### Kubernetes 구성 요소 설치

kubeadm, kubelet, kubectl을 설치합니다.

```bash
# 필요한 패키지 설치
sudo apt-get install -y apt-transport-https ca-certificates

# Kubernetes GPG 키 및 저장소 추가
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

# 패키지 목록 업데이트
sudo apt-get update --allow-unauthenticated

# Kubernetes 구성 요소 설치
sudo apt-get install -y --allow-unauthenticated kubelet kubeadm kubectl
```

<!-- Kubernetes 구성 요소 설치 확인 스크린샷 -->

## 🎯 마스터 노드 설정

### 클러스터 초기화

마스터 노드에서만 실행하는 클러스터 초기화 과정입니다.

```bash
# 클러스터 이미지 사전 다운로드
sudo kubeadm config images pull --cri-socket unix:///run/containerd/containerd.sock

# 마스터 노드 초기화
sudo kubeadm init \
  --apiserver-advertise-address=10.0.2.15 \
  --pod-network-cidr=192.168.0.0/16 \
  --cri-socket /run/containerd/containerd.sock
```

### kubectl 설정

마스터 노드에서 kubectl을 사용할 수 있도록 설정합니다.

```bash
# kubeconfig 설정
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### CNI(Calico) 설치

Pod 간 네트워크 통신을 위한 Calico CNI를 설치합니다.

```bash
# Calico 매니페스트 적용
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# CNI 파드 상태 확인
watch kubectl get pods -n kube-system
```

<img width="815" height="331" alt="image" src="https://github.com/user-attachments/assets/4e0d0428-59be-42d5-ba8b-148abe56dda5" />


### 워커 노드 조인 토큰 확인

```bash
# 조인 토큰 확인
sudo kubeadm token list

# 조인 명령어 전체 생성 (필요시)
kubeadm token create --print-join-command
```
<img width="1635" height="312" alt="image" src="https://github.com/user-attachments/assets/67826bc1-59db-4556-b43e-0a5b762aea72" />


## 🔧 워커 노드 설정

각 워커 노드에서 마스터 노드에 조인하는 과정입니다.

### 클러스터 조인

마스터 노드 초기화 시 출력된 조인 명령어를 실행합니다.

```bash
# 워커 노드에서 실행 (myserver02, myserver03)
sudo kubeadm join 10.0.2.15:6443 --token <토큰> \
  --discovery-token-ca-cert-hash sha256:<해시값>
```

### 워커 노드에서 kubectl 설정

워커 노드에서도 kubectl을 사용하려면 kubeconfig를 복사합니다.

```bash
# 마스터 노드의 config 파일 복사
mkdir -p $HOME/.kube
scp ubuntu@myserver01:~/.kube/config ~/.kube/config
```

## ✅ 클러스터 검증

### 노드 상태 확인

```bash
# 모든 노드가 Ready 상태인지 확인
kubectl get nodes -o wide
```

<!-- 클러스터 노드 Ready 상태 스크린샷 -->
<img width="1447" height="323" alt="image" src="https://github.com/user-attachments/assets/2d0ef923-859f-45b8-b273-6d9a589dc30f" />


### 시스템 파드 상태 확인

```bash
# kube-system 네임스페이스의 모든 파드 상태 확인
kubectl get pods -n kube-system
```

<img width="904" height="450" alt="image" src="https://github.com/user-attachments/assets/4fa20073-a66c-4950-b314-0cce8b7235fd" />


### 샘플 애플리케이션 배포

클러스터가 정상적으로 작동하는지 확인하기 위한 테스트 배포입니다.

```bash
# nginx 배포 생성
kubectl create deployment nginx1 --image=registry.k8s.io/nginx

# 모든 리소스 확인
kubectl get all

# NodePort 서비스로 노출
kubectl expose deployment nginx1 --type=NodePort --port=8081 --target-port=80

# 서비스 확인
kubectl get svc

# 파드 위치 확인
kubectl get pods -o wide
```

<img width="1107" height="570" alt="image" src="https://github.com/user-attachments/assets/9724af69-cb17-4c98-be62-37eeb76f2211" />


## 🌐 네트워크 설정 및 서비스 타입

### Ingress Controller 설치

외부 트래픽을 클러스터 내부로 라우팅하기 위한 NGINX Ingress Controller를 설치합니다.

```bash
# NGINX Ingress Controller 설치
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml

# Ingress Controller 파드 상태 확인
kubectl get pods -n ingress-nginx
```

### 서비스 타입별 테스트

```bash
# 추가 테스트 배포 생성
kubectl create deployment nginx2 --image=nginx
kubectl create deployment nginx3 --image=nginx

# ClusterIP 서비스 생성
kubectl expose deployment nginx2 --type=ClusterIP --port=8082 --target-port=80
kubectl expose deployment nginx3 --type=ClusterIP --port=8083 --target-port=80

# 모든 서비스 확인
kubectl get svc
```



## 🔍 트러블슈팅 가이드

### 일반적인 문제 해결

#### 1. CrashLoopBackOff 오류

CrashLoopBackOff는 파드가 지속적으로 실패하여 재시작하는 상태입니다.

**진단 방법:**
```bash
# 파드 상태 확인
kubectl get pods

# 파드 상세 정보 확인
kubectl describe pod <pod-name>

# 파드 로그 확인
kubectl logs <pod-name> --previous
```

**해결 방법:**
- 컨테이너 이미지 확인 및 수정
- 리소스 제한 조정
- 환경 변수 및 설정 검증

#### 2. ImagePullBackOff 오류

컨테이너 이미지를 가져올 수 없는 상태입니다.

**진단 방법:**
```bash
kubectl describe pod <pod-name>
kubectl events
```

**해결 방법:**
- 이미지 이름 및 태그 확인
- 레지스트리 인증 정보 확인
- 네트워크 연결 상태 점검

#### 3. NodeNotReady 오류

노드가 준비되지 않은 상태입니다.

**진단 방법:**
```bash
kubectl get nodes
kubectl describe node <node-name>
```

**해결 방법:**
- kubelet 서비스 상태 확인
- 노드 리소스 확인
- 네트워크 연결 문제 해결

### CNI 관련 문제 해결

#### CNI 플러그인이 초기화되지 않음

"CNI Plugin Not Initialized" 오류 발생 시:

```bash
# CNI 파드 상태 확인
kubectl get pods -n kube-system | grep calico

# CNI 로그 확인
kubectl logs -n kube-system <calico-pod-name>

# 네트워크 CIDR 설정 확인
kubectl cluster-info dump | grep -i cidr
```

**해결 방법:**
- pod-network-cidr 설정 확인
- Calico 매니페스트 재적용
- 노드 재시작

#### 파드 간 네트워크 통신 문제

```bash
# 파드 IP 할당 확인
kubectl get pods -o wide

# 네트워크 정책 확인
kubectl get networkpolicies --all-namespaces

# DNS 해결 테스트
kubectl run test-pod --image=busybox --rm -it -- nslookup kubernetes.default
```

### 클러스터 초기화 실패 해결

클러스터 초기화가 실패한 경우 완전 초기화 후 재시도:

```bash
# 클러스터 상태 완전 리셋
sudo kubeadm reset -f

# 관련 서비스 중지
sudo systemctl stop kubelet
sudo systemctl stop containerd

# 설정 파일 삭제
sudo rm -rf /etc/kubernetes /var/lib/etcd

# containerd 재시작
sudo systemctl start containerd

# 클러스터 재초기화
sudo kubeadm init --apiserver-advertise-address=10.0.2.15 \
  --pod-network-cidr=192.168.0.0/16 \
  --cri-socket /run/containerd/containerd.sock
```

### 성능 및 모니터링

#### 리소스 사용량 확인

```bash
# 노드 리소스 사용량
kubectl top nodes

# 파드 리소스 사용량
kubectl top pods --all-namespaces
```

#### 클러스터 이벤트 모니터링

```bash
# 전체 이벤트 확인
kubectl get events --sort-by=.metadata.creationTimestamp

# 특정 네임스페이스 이벤트 확인
kubectl get events -n kube-system
```

## 💡 운영 모범 사례

### 보안 강화

1. **RBAC 설정**: 역할 기반 액세스 제어 구현
2. **네트워크 정책**: Pod 간 통신 제한
3. **시크릿 관리**: 민감한 정보의 안전한 저장
4. **이미지 스캐닝**: 보안 취약점 검사

### 고가용성 구성

1. **다중 마스터 노드**: Control Plane 이중화
2. **Pod Disruption Budget**: 서비스 중단 최소화
3. **리소스 제한**: 적절한 CPU/메모리 할당
4. **백업 전략**: etcd 데이터 정기 백업

### 성능 최적화

1. **노드 어피니티**: 효율적인 파드 스케줄링
2. **Horizontal Pod Autoscaler**: 자동 스케일링 설정
3. **리소스 모니터링**: Prometheus/Grafana 구축
4. **네트워크 최적화**: CNI 성능 튜닝

## 🎉 결론

이 가이드를 통해 Ubuntu 24.04 LTS 환경에서 기능하는 Kubernetes 클러스터를 구축했습니다. 실제 운영 환경에서는 보안, 고가용성, 모니터링 등의 추가 고려사항을 반영하여 더욱 안정적인 클러스터 운영이 가능할 것으로 예상됩니다. 전체적인 k8s 구성과 아키텍처 흐름을 구현해 보면서 이해할 수 있었습니다.

---

## 📚 참고 자료

- [Kubernetes 공식 문서](https://kubernetes.io/docs/)
- [Calico 네트워킹 가이드](https://projectcalico.org/)
- [Ubuntu 서버 관리 가이드](https://ubuntu.com/server/docs)
- [VMware 가상화 베스트 프랙티스](https://docs.vmware.com/)

**다음 Repository**: [k8s-monitoring-guide](https://github.com/Gill010147/k8s-monitoring-guide)
