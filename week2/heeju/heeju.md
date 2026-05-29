## 쿠버네티스란 무엇인가?

**오케스트레이션**  
: 여러 개의 작업(자동화된 작업 포함)을 하나의 종단간 프로세스로 구성하고 조율하는 것

**클러스터**  
: 여러 대의 독립적인 서버를 네트워크로 연결해 하나의 고성능 시스템처럼 운영하는 방식

**쿠버네티스 클러스터**  
: 컨테이너화된 애플리케이션을 배포하고 관리하는 노드(컴퓨터)들의 집합
-> 쿠버네티스 전체 시스템 (서버 여러 대를 묶은 운영 플랫폼)

**노드**  
:클러스터 안에 들어있는 실제 컴퓨터

- Master Node
- Worker Node

![alt text](<쿠버네티스의 컴포넌트.svg>)

**Pod (파드)**  
: 쿠버네티스가 생성하고 관리하는 가장 작은 컴퓨팅 단위로, 한 개 이상의 리눅스 컨테이너로 구성되며 애플리케이션(의 인스턴스)이 실행되는 논리적 호스트(컴퓨터)
-> 실제 실행되는 앱, 컨테이너 실행 단위

쿠버네티스는 컨테이너를 직접 실행하지 않고, Pod라는 껍데기 안에서 실행함  
why? 네트워크, 스토리지, 라이프 사이클 관리... 이런 걸 Pod 단위로 묶는 거임

<pre>
Node(서버)
  ㄴ Pod
      ㄴ Docker Container (내 앱)
</pre>

![alt text](<쿠버네티스의 pod 운영.png>)

참고. https://computing-jhson.tistory.com/102

**watch**  
: 리소스의 변경 이벤트를 실시간으로 구독하는 메커니즘  
-> 현재 상태 한 번 조회하는 것이 아니라 앞으로 발생할 변경사항을 계속 전달받음  
ex) Pod를 watch하면 `생성됨(ADDED)`, `수정됨(MODIFIED)`, `삭제됨(DELETED)` 같은 이벤트를 API Server가 계속 보내줌

<pre>
kubectl get pods --watch
kubectl get pods --w
</pre>

- Pod 상태가 바뀔 때마다 실시간으로 출력이 갱신됨

Controller에서도 **변경 생길 때만 이벤트 전달**되기 때문에 1초마다 전체 조회하는 것보다 훨씬 가볍고 실시간성이 좋음

<대표 이벤트 타입>  
이벤트 | 의미
-- | --
ADDED | 리소스 생성
MODIFIED | 리소스 변경
DELETED | 리소스 삭제
BOOKMARK | 현재 동기화 지점 표시
ERROR | watch 오류

**리스 (Lease)**  
: Kubernetes에서 특정 주체가 일정 시간 동안 어떤 역할이나 권한을 가지고 있음을 기록하는 매우 가벼운 리소스 오브젝트  
-> “현재 누가 담당 중인지”, “누가 아직 살아있는지”를 저장하는 용도로 사용  
=> Kubernetes 내부에서 “현재 활성 상태인 주체” 또는 “현재 작업 담당자”를 효율적으로 추적하기 위한 시간 기반 상태 기록 리소스

- 현재 권한을 가진 대상(holderIdentity)과 마지막 갱신 시간(renewTime) 등 기록
- 일정 시간 동안 갱신되지 않으면 권한이 만료된 것으로 판단

ex)

- **Leader Election**
  - 동일한 컨트롤러 Pod가 여러 개 실행 중일 때, 모든 Pod가 동시에 같은 작업을 수행하면 중복 처리나 충돌이 발생할 수 있다. 이를 방지하기 위해 Kubernetes는 하나의 Pod만 leader로 선택하여 실제 관리 작업(Reconcile)을 수행하게 한다. 이때 Lease 오브젝트에 현재 leader 정보를 기록한다. Leader Pod는 주기적으로 Lease를 갱신하며 자신의 생존 상태와 권한 유지 여부를 알린다. 만약 leader Pod가 종료되거나 장애가 발생해 Lease 갱신이 멈추면, 다른 Pod가 Lease를 획득하여 새로운 leader가 된다.

- Node Heartbeat 용도
  - kubelet은 주기적으로 Lease를 갱신하여 “이 노드는 아직 정상 동작 중이다”라는 정보를 Kubernetes Control Plane에 전달한다. Control Plane은 Lease 갱신이 일정 시간 동안 멈추면 해당 Node가 비정상 상태라고 판단할 수 있다.

## 클러스터 아키텍처

![alt text](<쿠버네티스 클러스터 구조.svg>)

**EndpointSlice 오브젝트**  
: Service가 연결해야 하는 Pod들의 IP/포트 정보를 저장하는 쿠버네티스 리소스  
-> 이 Service 뒤에 실제로 연결될 Pod들이 누구인지 적어놓은 목록

예전에는 Endpoints라는 오브젝트 하나에 다 저장했지만, Pod가 수백~수천 개가 되면 너무 커져서 비효율적이었음. 그래서 나온 게 **EndpointSlice다**! 엔드포인트를 여러 조각(slice)으로 나눠 관리하는 방식임. => Service가 트래픽 보낼 Pod 주소 목록을 효율적으로 저장하는 객체

**Service**  
: Pod들이 바뀌어도 고정된 접근 주소를 제공하는 쿠버네티스 리소스

**ServiceAccount**  
: Pod 안의 애플리케이션이 쿠버네티스 API를 사용할 때 쓰는 계정

쿠버네티스는 기본적으로 누가 요청했는지, 어떤 권한이 있는지 확인하기 때문에 Pod나 컨테이너도 쿠버네티스 API를 사용하려면 자신이 누구인지 알려주는 과정이 필요함

**장애 허용성(Fault Tolerance)**
: 일부가 고장나도 전체 시스템은 계속 동작할 수 있는 능력

**정적 파드(Static Pod)**  
: API 서버가 아니라 특정 노드의 kubelet이 직접 관리하는 파드

컨트롤 플레인 통해 생성하는 일반 Pod가 아니라 노드에 직접 적어놓고 kubelet이 실행시키는 Pod임

**허브 앤 스포크(Hub-and-Spoke)**
: 중앙 허브(Hub)를 기준으로 여러 지점(Spoke)이 연결되는 구조
-> 쿠버네티스의 API 패턴

**RBAC (Role-Based Access Control)**  
: 역할(Role) 기준으로 권한을 관리하는 방식 => 누가 뭘 할 수 있는지 역할별로 제한하는 인가 시스템

쿠버네티스에서는 이 사용자/Pod가 무슨 작업 가능한가를 RBAC로 제어함

**ClusterRole**  
: 클러스터 전체 범위 권한

**페이지 캐시(Page Cache)**  
: 디스크 파일 내용을 메모리에 임시 저장해두는 캐시  
-> 파일을 디스크에서 매번 읽으면 느리니까 RAM에 잠깐 저장해두는 것

**라이트백(Write Back)**  
: 데이터를 바로 디스크에 쓰지 않고, 일단 메모리 캐시에 저장한 뒤 나중에 디스크에 기록하는 방식

**Deployment**  
: 일반적인 무상태(stateless) 앱 Pod들을 관리하는 리소스

<기능>

- Pod 자동 생성
- 롤링 업데이트
- 자동 복구
- 스케일링

**StatefulSet**  
: 상태(state)가 중요한 앱 관리용 리소스

ex) DB, Kafka, Redis Cluster

**퍼시스턴트볼륨 컨트롤러(PersistentVolume Controller)**  
: 저장소(볼륨)를 Pod에 연결하거나 해제하는 걸 관리하는 컨트롤러  
-> Pod가 사용할 디스크를 붙였다 떼는 관리자

**Persistent Volume (PV, 퍼시스턴트 볼륨)**
: 쿠버네티스에서 관리하는 영구 저장공간

ex) SSD, EBS, NFS, 클라우드 디스크
