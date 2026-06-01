# CSI
- 옛날 k8s: 새 스토리지 추가할려면 k8s 전체 코드 수정 + 재배포 (in-tree 구조)
- CSI 이후: 스토리지 = 외부 플러그인 (독립 배포)
<img width="547" height="378" alt="image" src="https://github.com/user-attachments/assets/e3e62e10-6bed-4940-b897-28b8c4c5816f" />


## gRPC 기반 통신
- CSI driver는 gRPC 서버로서 아래 3가지의 주요 인터페이스를 구현해 응답함
  - Identity Service: 플러그인의 이름(예: rbd.csi.ceph.com), 버전 정보 등을 k8s에게 알려줌
  - Controller Service: 볼륨 생성/삭제, 스냅샷 생성, 볼륨 확장 등 클러스터 전역에서 발생하는 논리적인 스토리지 관리 작업을 담당
  - Node Service: 실제 워커 노드에서 스토리지 장치를 포맷하거나 파일시스템에 마운트/언마운트 작업 담당하며 모든 워커 노드에 하나씩 설치되어 실행됨

참고)
```
사용자 의도          k8s 추상화           실제 동작
─────────────────────────────────────────────────
PVC 생성     →    "10Gi 블록 스토리지 줘"
                       ↓
                  PV 바인딩         →   CreateVolume (Controller)
                                        Ceph에 RBD 이미지 생성
                       ↓
                  파드 스케줄링
                       ↓
                  Attach            →   ControllerPublishVolume
                                        볼륨을 노드에 연결
                       ↓
                  Mount (2단계)     →   NodeStageVolume
                                        + NodePublishVolume

```
 
- mount가 두 단계로 나누어진 이유 (mount namespace를 통해 파일시스템 마운트 지점을 격리함)
  - NodeStageVolume: 실제 스토리지(ceph 등..)를 워커 노드에 최초 연결하고 파일 시스템을 포맷한 뒤 노드의 전역 디렉토리에 마운트함
  - NodePublishVolume: 노드의 글로벌 디렉토리를 각 pod가 사용하는 전용 디렉토리에 bind mount 함
- 즉, 파드가 10개 뜰때 디스크 연결 작업과 포맷을 10번 할 필요 없이 bind mount만 해주면 됨

### Bind Mount
- 일반적 mount는 C드라이브 등 물리적인 디스크를 특정 폴더에 연결하는 것을 의미함
- Bind mount는 디스크가 아니라, 이미 존재하는 폴더나 파일을 다른 경로에서 접근할 수 있게 해주는 말함
  - 원본 폴더와 대상 폴더가 물리적으로 같은 디스크 공간(inode)을 가리키게 만들음 (접근 통로를 여러개 두는 방식)
<img width="561" height="354" alt="image" src="https://github.com/user-attachments/assets/13c29fba-9a07-4ad4-8d91-ec588cb88601" />


### Unix Domain Socket
- IPC 방식 중 하나를 나타냄
- 네트워크 port 대신 리눅스 소켓 파일 시스템(ex: csi.sock) 형태로 존재함
- 속도가 TCPI/IP 통신보다 빠르며, 파일 권한을 가지기에 보안이 좋음

### k8s CSI Sidecar Container
- ceph같은 스토리지들이 k8s 내부를 알 필요없이 gRPC 인터페이스만 바라보도록 대리 역할을 수행해줌
- 즉, 스토리지가 k8s API를 직접 다루는 복잡한 코드를 작성할 필요가 없음
- 주요 CSI 사이드카 컨테이너들의 종류와 역할에 대해 알아보자
1. csi-provisioner (External Provisioner)
- 하는 일: 사용자가 PVC(PersistentVolumeClaim)를 생성하는지 감시함

통신 중계: 새로운 PVC가 감지되면, 사이드카가 csi.sock을 통해 Ceph CSI 드라이버에게 gRPC로 CreateVolume 명령을 내리며, 반대로 PVC가 삭제되면 DeleteVolume을 호출하여 실제 Ceph 클러스터에서 스토리지를 지움

2. csi-attacher (External Attacher)
하는 일: 생성된 볼륨을 특정 워커 노드에 물리적/논리적으로 붙이는 작업을 감시하며, 쿠버네티스의 VolumeAttachment 객체를 감시함

통신 중계: 볼륨을 노드에 연결해야 할 때 Ceph CSI 드라이버의 ControllerPublishVolume을 호출하고, 해제할 때는 ControllerUnpublishVolume을 호출함

3. csi-node-driver-registrar (Node Driver Registrar)
하는 일: 이 사이드카는 각 워커 노드마다 하나씩 실행됨
- 워커 노드에서 대기 중인 Kubelet에게 해당 노드에 Ceph CSI 드라이버가 새로 생겼으니 앞으로 볼륨 마운트는 이 드라이버한테 하라고 등록해 주는 역할을 함

통신 중계: Kubelet의 플러그인 디렉터리에 소켓을 연결하여 CSI 드라이버의 신원 정보를 전달함

4. csi-resizer (External Resizer)
하는 일: 사용자가 이미 사용 중인 PVC의 용량을 늘렸을 때를 감시함

통신 중계: 용량 변경을 감지하면 CSI 드라이버에게 gRPC로 ControllerExpandVolume 명령을 보내 실제로 Ceph의 RBD나 CephFS 볼륨 크기를 늘려줌

5. csi-snapshotter (External Snapshotter)
하는 일: 쿠버네티스에서 VolumeSnapshot 요청이 들어오는지 감시함

통신 중계: 요청이 들어오면 CSI 드라이버에게 CreateSnapshot 또는 DeleteSnapshot gRPC 명령을 내려서 데이터의 백업본(스냅샷)을 제어함

### Provision
<img width="360" height="467" alt="image" src="https://github.com/user-attachments/assets/9663e1f6-14a4-4b9f-9336-fea242a91c9b" />

### Attach
<img width="400" height="461" alt="image" src="https://github.com/user-attachments/assets/7fb16c06-e383-463e-825e-d51c9276e010" />

### Mount
<img width="298" height="456" alt="image" src="https://github.com/user-attachments/assets/3fbb09cd-820e-442b-94fd-e6cbe3fd7a00" />

