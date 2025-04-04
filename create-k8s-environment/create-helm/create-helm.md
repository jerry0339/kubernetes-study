# Helm

## 1. Helm 설치
* k8s `1.27.x - 1.30.x` 버전 기준 helm `3.15.x`버전이 호환이 잘 되는 버전임
* 3.15.4 버전으로 설치
```sh
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list

sudo apt-get update
sudo apt-cache madison helm # 설치 가능한 버전 확인
sudo apt-get install -y helm=3.15.4-1
helm version # 설치 잘 되었는지 확인
```

## helm 관련 내용들
* [Helm 문서](/helm/helm.md)
* [jenkins 문서](/CICD/ci-jenkins/jenkins.md)
* [argocd 문서](/CICD/cd-argocd/argocd.md)