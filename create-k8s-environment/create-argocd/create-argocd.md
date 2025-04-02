# Argo CD
* Master 노드에서 Helm으로 ArgoCD 설치하기

<br>

## 1. Helm Repository 추가
```sh
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

<br><br>

## 2. 네임스페이스 생성
```sh
kubectl create namespace argocd
```

<br><br>

## 3. ArgoCD 설치
* NodePort type으로 설치
    ```sh
    # values.yaml 파일 생성
    cat > argocd-values.yaml <<EOF
    server:
      service:
        type: NodePort
        nodePortHttps: 32000
    EOF

    # helm으로 설치
    helm install argocd argo/argo-cd -f argocd-values.yaml -n argocd
    ```
* ingress 컨트롤러 배포된 경우, LoadBalancer type으로 설치
    ```sh
    # values.yaml 파일 생성
    cat > argocd-values.yaml <<EOF
    server:
      service:
        type: LoadBalancer
      ingress:
        enabled: true
    EOF

    # helm으로 설치
    helm install argocd argo/argo-cd -f argocd-values.yaml -n argocd
    ```

<br><br>

## 4. LoadBalancer type 으로 변경
* ingress 컨트롤러가 배포되어 있어야 함
* 3에서 NodePort로 설치하고나서 LoadBalancer type 으로 변경하고자 하는 경우
    ```sh
    # values.yaml 파일 생성
    cat > argocd-values.yaml <<EOF
    server:
      service:
        type: LoadBalancer
      ingress:
        enabled: true
    EOF

    # Helm 업그레이드 (helm으로 argocd 설치된 경우에만, 없을 경우 helm install 명령어 사용)
    helm upgrade argocd argo/argo-cd -f argocd-values.yaml -n argocd

    # 업데이트 내용 확인
    kubectl get svc argocd-server -n argocd
    ```

<br><br>

## 5. 대시보드 접속하기
* NodePort type 설정시, `https://{도메인}:32000`으로 접속
* LoadBalancer type 설정시, Ingress 컨트롤러가 배포되어 있어야 하며 SSL/TLS 설정이 없으면 HTTP 기본 포트(80)로만 접근 가능함 주의
* ArgoCD 대시보드 admin 계정 비밀번호 조회
    ```sh
    # 대시보드 초기 비밀번호 조회
    kubectl -n argocd get secret argocd-initial-admin-secret \
      -o jsonpath="{.data.password}" | base64 -d; echo
    ```
* ID는 admin으로, 위에서 조회한 비밀번호를 입력하고 로그인
* ![](2025-04-03-00-38-29.png)

