## 사전 작업
* vpc port open 
  * internal open - 0.0.0.0/0
  * master open - 6443
  * worker open - 26443, 30000-32767
* kubeadm join 명령어 확인

<br>

## 스크립트
```sh
#!/bin/bash
set -e  # 에러 발생 시 스크립트 중단

echo "🔧 Kubernetes 설치 시작..."
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# 네트워크 모듈 로드
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl 파라미터 설정
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# sysctl 파라미터 적용
sudo sysctl --system

sudo apt-get update
sudo apt install -y containerd

sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml > /dev/null

# SystemdCgroup = true 로 자동 수정
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd.service

sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add the new repository (v1.30)
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet=1.30.13-1.1 kubeadm=1.30.13-1.1 kubectl=1.30.13-1.1
echo "✅ Kubernetes 설치 완료"

echo "🔧 Kubernetes Cluster에 worker 노드 추가 중..."
sudo kubeadm join k8s-master.flowchat.shop:6443 --token rdtrtb.d4zykse63xh6ahbs \
        --discovery-token-ca-cert-hash sha256:5c257af517639a2ba39c83ac812f9028d1f7db24bfe33ceafb0a2b642787b6f1
echo "✅ Kubernetes Cluster에 worker 노드 추가 완료"
```