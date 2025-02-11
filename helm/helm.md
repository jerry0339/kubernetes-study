# Helm
* 참고 : https://github.com/k8s-1pro/kubernetes-anotherclass-sprint2

## helm 설치
* 3.13.2 버전 설치
```sh
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list

sudo apt-get update
sudo apt-cache madison helm # 설치 가능한 버전 확인
sudo apt-get install -y helm=3.13.2-1
helm version # 설치 잘 되었는지 확인
```

<br>

## Helm 패키지 간단 설명
* ![](2025-02-04-16-55-28.png)

<br>

## Helm Template 사용법 간단한 설명
* `helper.tpl`에 들어가는 내용은 yaml파일에 모두 적용 가능함
  * include 명령어 뒤의 내용에 들어감
  * ex. `{{ include "api-tester.fullname" . }}`
* Chart.yaml, values.yaml 파일
  * 모든 yaml에서 공통으로 가져다 쓸 수 있음
  * .Chart 또는 .Values 명령어 작성하면 됨
  * ex. `{{ .Values.replicaCount }}`, `{{ .Chart.Name }}`
* Helm Template에서 동적 데이터 작성 방법
  * 예시 `{{- toYaml .Values.configmap.data.properties | nindent 2 }}`
  * `{{}}` : 안에 동적 데이터에 대한 내용을 작성하면 해당하는 위치에 데이터가 작성됨
  * `-` : 해당 위치에 공백 데이터가 있으면 모두 삭제해 줌
  * `.Values.configmap.data.properties` : values.yaml 파일의 configmap.data.properties 데이터가 위치하게 됨
  * `nindent 2` : 데이터 앞에 공백 2개가 추가됨
  * `toYaml` : Helm 템플릿에서 배열 같은 객체(예: 배열, 딕셔너리, 중첩된 데이터 구조 등)를 YAML 형식의 문자열로 변환
    * `{{- toYaml .Values.configmap.data.properties | nindent 2 }}`라는 데이터는 아래와 같이 두줄로 변경됨
    ```yaml
    # values.yaml의 아래의 데이터가
    configmap:
      data:
        properties:
          key1: value1
          key2: value2
    ```
    ```yaml
    # 아래와 같이 변경되어 {{- toYaml .Values.configmap.data.properties | nindent 2 }} 위치에 추가됨, 공백 2칸 확인
      key1: value1
      key2: value2
    ```