# zRAM, Swap 메모리 설정

## zRAM & Swap 메모리란?
* `zRAM`: RAM의 일부를 압축 블록 디바이스로 만들어 swap 공간처럼 사용하는 기술
  * 실제로 디스크(SSD/HDD)에 쓰지 않고, 압축된 데이터를 RAM에 저장
  * RAM을 더 효율적으로 활용할 수 있고, swap I/O가 매우 빠름
  * 압축/해제에 CPU 자원을 소모하지만, 디스크 swap보다 훨씬 빠름
* `Swap 메모리`: 디스크 기반 스왑으로, swap은 디스크(SSD/HDD)에 할당된 공간에 메모리 페이지를 저장
  * 주요 목적 - RAM이 부족할 때 데이터를 디스크로 옮겨, 시스템이 **OOM(Out of Memory)** 없이 동작할 수 있도록 함
  * 디스크는 RAM보다 매우 느리기 때문에, swap 사용이 많아지면 시스템 속도가 크게 저하
    * Swap 메모리 설정만 해도 엄청 느려짐
* zRAM만 사용?
  * zRAM만 사용하면, swap 공간이 RAM에만 생기므로 RAM이 모두 소진되면 OOM이 발생함
  * OOM 방지가 목적이라면 Swap 메모리도 함께 사용헤야 함
* zRAM와 swap 메모리 같이 사용하는 경우?
  * 해당 경우에는, zRAM이 먼저 사용되고 zRAM이 가득 차면, 그 다음에 디스크 swap이 사용됨

<br>

## zRAM 설정 방법
* 1.5GB(1536M) 할당 예시
  ```sh
  sudo modprobe zram num_devices=1
  echo lz4 | sudo tee /sys/block/zram0/comp_algorithm
  echo 1536M | sudo tee /sys/block/zram0/disksize
  sudo mkswap /dev/zram0
  sudo swapon /dev/zram0 -p 10
  ```
* zRAM은 재부팅시 초기화되니 주의

<br>

## Swap 메모리 설정
* 2GB 할당 예시
  ```sh
  sudo fallocate -l 2G /swapfile
  sudo chmod 600 /swapfile
  sudo mkswap /swapfile
  sudo swapon /swapfile
  ```
* cf. 기존 swap 비활성화 및 삭제
  ```sh
  sudo swapoff -v /swapfile  # swap 비활성화
  sudo rm /swapfile          # swap 파일 삭제
  ```

<br>

## 적용 확인
```sh
zramctl # zRAM 상태 확인
swapon --show # swap 메모리 적용 확인
free -h # 메모리 전체 사용량 확인 - (zRAM + Swap 메모리)로 나옴
```