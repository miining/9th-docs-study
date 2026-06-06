## 서비스, 로드밸런싱, 네트워킹

**서비스 (Service)**  
: 파드들의 고정된 접속 주소를 제공하는 오브젝트  
파드는 비영구적 리소스이며, 자꾸 죽고 새로 만들어지기 때문에 IP가 바뀐다. 그래서 서비스가 등장했다. (고정 IP, 로드밸런싱 제공)

<pre>
Client
   ↓
Service
   ↓
Pod A
Pod B
Pod C
</pre>

서비스는 고정된 IP를 갖고 있고, 뒤에 있는 파드가 바뀌어도 서비스가 알아서 현재 살아있는 파드로 트래픽을 보낸다.

**kube-proxy**  
: 클러스터의 각 노드에서 실행되는 네트워크 프록시로, 각 노드에서 Service 트래픽을 실제 Pod로 전달하기 위한 네트워크 규칙을 관리한다.
-> 쿠버네티스 Service 추상화를 실제 네트워크 포워딩 규칙으로 변환하는 역할을 한다.

**헤드리스 서비스 (Headless Service)**  
: Service인데 가상 IP와 로드밸런싱은 없고, DNS를 통해 실제 Pod들의 주소를 직접 알려주는 서비스  
-> 주로 StatefulSet, 데이터베이스 클러스터, Kafka 같은 상태 저장 서비스에서 사용한다.

로드밸런싱 없이 개별 Pod를 직접 알고 싶을 때 사용한다. 셀렉터가 있는 경우 쿠버네티스가 알아서 Pod들을 찾아서 DNS에 등록하지만, 없는 경우네는 사용자가 직접 EndpointSlice를 만들어서 서버들을 등록해야 한다.

**인그레스 (Ingress)**  
: 클러스터 외부에서 들어오는 HTTP/HTTPS 요청을 어떤 Service로 보낼지 정하는 규칙

서비스만 쓰면 서비스마다 로드밸런서를 하나씩 만들어야 해서 비효율적이지만, Ingress를 쓰면 하나의 진입점으로 여러 서비스를 연결할 수 있다.

<pre>
인터넷
   ↓
Ingress
 ├─ /api    → Service A
 ├─ /admin  → Service B
 └─ /blog   → Service C
</pre>

이렇게 하나의 Ingress가 여러 Service로 요청을 분기하는 것을 인그레스 팬아웃(Fanout)이라고 한다.

![alt text](<인그레스 팬아웃.png>)

**TLS (Transport Layer Security)**  
: 인터넷 통신을 암호화해서 중간에서 내용을 못 보게 하는 보안 프로토콜

TLS가 없이 데이터를 전송하면, 중간에서 패킷 확인 시 데이터가 그대로 보인다. TLS 사용 시에는 암호화된 데이터만 보인다.

쿠버네티스에서는 Ingress가 TLS 인증서를 사용해 HTTPS를 처리한다.

**이그레스 (Egress)**  
: 파드가 밖으로 나가는 트래픽

<pre>
A → B (A가 B로 요청)

A 입장 = Egress
B 입장 = Ingress
</pre>

**파드 격리 (Isolation)**  
: 네트워크 정책(NetworkPolicy)에 의해 제한을 받는 상태
기본적으로 쿠버네티스에서는 모든 파드가 서로 자유롭게 통신할 수 있다.

- Ingress 격리를 적용하면 해당 파드로 들어오는 트래픽이 제한되어, 정책에서 허용한 파드·네임스페이스·IP만 해당 파드에 요청할 수 있다.
- Egress 격리를 적용하면 해당 파드에서 나가는 트래픽이 제한되어, 정책에서 허용한 대상에게만 요청을 보낼 수 있다.

따라서 파드 간 통신이 성공하려면 송신 파드의 Egress 정책과 수신 파드의 Ingress 정책이 모두 해당 통신을 허용해야 한다.

**토폴로지(Topology)**  
: 네트워크나 인프라의 물리적/논리적 구조. 여기서는 주로 어떤 리전(region), 존(zone)에 리소스가 있는지를 의미

**토폴로지 인지 힌트(Topology Aware Hints)**  
: 서비스의 엔드포인트(파드) 목록에 어느 존(Zone)에서 이 파드를 사용하는 것이 좋은지에 대한 정보를 추가하는 기능

<pre>
Zone-A
 ├─ Pod1
 └─ Pod2

Zone-B
 ├─ Pod3
 └─ Pod4
</pre>

클러스터가 이렇게 두 개의 존으로 구성되어 있을 때, 서비스 요청을 보내는 다른 파드가 Zone-A에 있다면 Pod1, Pod2를 우선 사용하게 한다. 네트워크가 다른 AZ를 왔다 갔다 하게 되면 지연시간이 늘어나고, 네트워크 비용도 늘어나고, 가용성도 줄어들기 때문!

## 스토리지

**Volume (볼륨)**  
:Pod가 사용하는 저장공간으로, 컨테이너가 재시작돼도 데이터가 유지될 수 있게 해준다.

**Mount (마운트)**  
:저장소를 특정 경로에 연결하는 것

<pre>
volumeMounts:
  - mountPath: /data
</pre>

- 컨테이너 안의 `/data` 폴더가 실제 볼륨과 연결된다.

**Ephemeral Volume (임시 볼륨)**  
: Pod가 살아있는 동안만 존재하는 볼륨 ex) emptyDir

**Persistent Volume (영구 볼륨)**  
: Pod가 죽어도 데이터 유지하는 볼륨 ex) EBS, NFS, PersistentVolume, PersistentVolumeClaim

**PV (PersistentVolume)**  
실제 저장 공간 (쿠버네티스 클러스터 입장에서 존재하는 디스크) ex) 100GB EBS, 500GB NFS, 1TB SSD

**PVC (PersistentVolumeClaim)**  
: 사용자가 저장 공간이 필요해요!라고 요청하는 객체

<pre>
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  resources:
    requests:
      storage: 50Gi
</pre>

- 50B 저장 공간 주라는 뜻

#### PV와 PVC의 관계

<pre>
PV (100GB)
      ↑
    바인딩
      ↓
PVC (50GB 요청)
</pre>

PVC(요청)가 생성되면 쿠버네티스가 적당한 PV(저장 공간)를 찾아 연결한다.

**StorageClass**  
: PV를 어떻게 만들지 정의하는 설계도

**동적 프로비저닝 (Dynamic Provisioning)**
: PVC 만들면 PV를 자동 생성하는 기능

<pre>
PVC 생성
 ↓
StorageClass 확인
 ↓
EBS 자동 생성
 ↓
PV 자동 생성
 ↓
PVC 연결
</pre>

**반환 정책 (Reclaim Policy)**  
: PVC 삭제 후 PV를 어떻게 처리할지 결정

Retain: 보존 (PV, 데이터 유지)
Delete: 삭제 (PV, 실제 디스크 삭제)

**Volume Plugin**  
: 쿠버네티스가 저장소와 통신하는 드라이버 ex) NFS, CSI, EBS, Ceph, iSCSI  
요즘은 CSI가 스토리지 표준이라고 하네요...

**접근 모드(Access Mode)**  
: PV/PVC에서 한 볼륨을 몇 개의 노드/파드가 어떻게 접근할 수 있냐를 나타냄

| 약어 | 원본             | 의미                                              |
| ---- | ---------------- | ------------------------------------------------- |
| RWO  | ReadWriteOnce    | 하나의 노드에서 읽기/쓰기 가능                    |
| ROX  | ReadOnlyMany     | 여러 노드에서 읽기만 가능                         |
| RWX  | ReadWriteMany    | 여러 노드에서 읽기/쓰기 가능                      |
| RWOP | ReadWriteOncePod | 클러스터 전체에서 단 하나의 파드만 읽기/쓰기 가능 |

**CSI (Container Storage Interface)**
: 쿠버네티스와 스토리지 업체를 연결하는 표준 인터페이스

예전에는 쿠버네티스가 직접 AWS EBS, GCE PD, Azure Disk 같은 스토리지를 내부 코드로 지원했지만, 관리가 힘든 문제로 인해 CSI가 등장했다.

<pre>
Kubernetes
      ↓
     CSI
      ↓
AWS EBS Driver
Azure Driver
Ceph Driver
NFS Driver
</pre>

이제 스토리지 업체가 CSI 드라이버만 만들면 된다.

**볼륨 헬스 모니터링 (Volume Health Monitoring)**  
: 스토리지 상태 감시 기능

<pre>
스토리지 장애
      ↓
CSI Driver 감지
      ↓
Kubernetes 알림
      ↓
PVC Event 생성
</pre>
