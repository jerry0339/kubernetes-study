# k8s trouble-shooting

## 1. Kubernates Evicted Status Pod 삭제하기
* 디스크 메모리 부족으로 Evicted Pod가 다수 생김
* Evicted Status인 pod를 한번에 지우는 스크립트
  ```sh
  kubectl -n default delete pods --field-selector=status.phase=Failed
  ```

<br><br>

## 2. Eviction Threahold 조정하기
* [참고링크](https://velog.io/@langoustine/%EC%BF%A0%EB%B2%84%EB%84%A4%ED%8B%B0%EC%8A%A4-Eviction-Threahold-%EC%A1%B0%EC%A0%95)
* 디스크 용량 확인
  * `df -h`
* 루트 파일 시스템의 디스크 공간이 86%로 Eviction Threshold 기본값인 85%를 초과하여 evicted pod가 계속해서 생기는 상황
* kubelet config 파일 위치 확인
  ```sh
  ps -ef | grep "kubelet --" | grep -v grep
  ```
* kubelet 설정파일에 원하는 Eviction Threshold 값을 기입하여 반영
  ```sh
  # write 권한 추가
  sudo chmod 666 /var/lib/kubelet/config.yaml
  # config파일 일반적인 위치
  sudo vi /var/lib/kubelet/config.yaml

  # config.yaml 파일의 마지막 줄에 아래의 내용 추가
  evictionHard:
    imagefs.available: 5% # 사용 가능한 디스크 공간이 전체의 5% 미만이면 DiskPressure 상태를 True로 활성화 (default: 15%)
    memory.available: 50Mi # 노드의 사용 가능한 메모리가 50MB 미만이면 MemoryPressure 상태를 True로 활성화 (default: 100MB)
    nodefs.available: 5% # 사용 가능한 디스크 공간이 전체의 5% 미만이면 DiskPressure 상태를 True로 활성화 (default: 10%)
    nodefs.inodesFree: 5% # 노드의 파일 시스템에서 사용 가능한 inode가 전체의 5% 미만이라면 DiskPressure 상태를 True 상태로 활성화 (default: 5%)

  # 위 내용 추가후 권한 복구
  sudo chmod 644 /var/lib/kubelet/config.yaml

  # kubelet 재시작, status 확인
  sudo systemctl restart kubelet
  sudo systemctl status kubelet
  ```
  ```sh
  # 복붙용
  evictionHard:
    imagefs.available: 5%
    memory.available: 50Mi
    nodefs.available: 5%
    nodefs.inodesFree: 5%
  ```