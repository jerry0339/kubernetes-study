## 사전 작업
* vpc port open 
  * internal open - 0.0.0.0/0
  * master open - 6443
  * worker open - 26443, 30000-32767
* `k8s-master` 하위 도메인 설정 확인
* kubeadm init - private IP / public IP or 도메인 주소 확인

<br>

## 스크립트
```sh
#!/bin/bash
set -e  # 에러 발생 시 스크립트 중단

echo "🔧 Kubernetes Master 노드 설정중..."
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

sudo kubeadm init \
  --control-plane-endpoint=k8s-master.flowchat.shop:6443 \
  --apiserver-advertise-address=10.128.0.16 \
  --pod-network-cidr=192.168.0.0/16 \
  --apiserver-cert-extra-sans=k8s-master.flowchat.shop,10.128.0.16

# kubeconfig 설정
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
echo "✅ Kubernetes Master 노드 설정 완료"


echo "🔧 Calico 설치중..."
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.3/manifests/calico.yaml

echo "✅ Calico 설치 완료"
```