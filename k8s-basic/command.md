# kubectl command

~~~bash
kubectl api-resources # 전체 api 리소스 확인
kubectl get all | grep {검색어}
kubectl get {타입1},{타입2}
kubectl get {타입} {리소스이름} -o wide
kubectl get {타입} {리소스이름} -o json
kubectl describe {타입} {리소스이름} # 리소스 상세 조회
kubectl delete {타입} {리소스이름}
kubectl logs {파드 이름}
kubectl logs -f {파드 이름} # 실시간 로그
kubectl logs {파드 이름} -c {컨테이너 이름} # 하나의 Pod에 여러개 컨테이너 있는 경우
kubectl patch {타입} {리소스이름} -p '{"spec" : {"suspend" : false }}' # -p옵션으로 JSON 또는 YAML 형식으로 전달된 패치를 적용

~~~