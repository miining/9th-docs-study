## 컨테이너

**RuntimeClass**  
: 이 Pod를 어떤 컨테이너 런타임 설정으로 실행할지 정의하는 리소스  
특수 런타임 사용 시 RuntimeClass를 씀
그런데 특정 런타임은 특정 노드에서만 동작 가능할 수 있기 때문에 해당 노드에 taint를 걸어둔다.

**taint**  
: 이 노드에는 아무 Pod나 오지말라고 노드에 표시해두는 기능

그렇지만 해당 RuntimeClass를 사용하는 Pod는 허용해야 하기 때문에...

**toleration**
: 이 RuntimeClass 쓰는 Pod는 taint 허용 가능하게 자동 설정

특수 노드는 아무 Pod나 못 오게 막아두고, 특정 RuntimeClass 사용하는 Pod만 예외적으로 허용함!

**훅(Hook)**  
: 특정 이벤트가 발생했을 때 자동으로 실행되도록 걸어두는 코드

**Lifecycle Hook**  
: 컨테이너 시작 직후, 컨테이너 종료 직전 같은 이벤트가 발생할 때 자동으로 실행되는 코드를 등록하는 것

**선점(preemption)**  
: 우선순위가 높은 Pod를 위해 기존 Pod를 강제로 제거하는 것

## 쿠버네티스에서의 윈도우

Kubernetes는 원래 리눅스 기반이지만, Windows Server 노드를 추가해서 윈도우 컨테이너도 같이 관리할 수 있음

**컨트롤 플레인 (Control Plane)**  
: Kubernetes 클러스터 전체를 관리하고 지시하는 중앙 관리 영역 (클러스터의 뇌)  
반대로 Worker Node는 실제로 컨테이너를 실행하는 부분이다!

## 워크로드

**논리 호스트(Logical Host)**  
: Pod를 하나의 “가상 컴퓨터”처럼 보는 개념
같은 Pod 안 컨테이너들은 같은 컴퓨터 안에서 함께 동작하는 것처럼 행동함

**cgroup(Control Group)**  
: 프로세스의 CPU·메모리 사용량 제한 기능
컨테이너별 리소스 제한에 사용됨

**수평 확장(Horizontal Scaling)**  
: 서버 성능을 높이는 대신 Pod 개수를 늘리는 방식 ex) Pod 1개 → 3개

**바인딩(Binding)**  
: Pod를 특정 Node에 연결하는 작업 (이 Pod는 이 Node에서 실행해라)

**세마포어(Semaphore)**  
: 여러 프로세스가 동시에 같은 자원 쓰지 못하게 순서를 조절하는 도구

**사이드카 컨테이너(Sidecar Container)**  
: 메인 컨테이너를 보조하는 컨테이너 (로그 수집, 프록시 등에 사용)  
-> 자체적으로 독립된 라이프사이클을 가짐

**고가용성(High Availability)**  
: 일부 서버가 죽어도 서비스가 계속 동작하는 구조

**하이퍼바이저(Hypervisor)**  
: VM(가상 머신)을 실행·관리하는 소프트웨어  
ex) VMware, Hyper-V

**커널 패닉(Kernel Panic)**
: 운영체제 커널이 치명적 오류로 멈춘 상태 (블루스크린 느낌)

**축출(Eviction)**  
: Kubernetes가 Pod를 강제로 노드 밖으로 내보내는 것

**노드 드레이닝(Node Draining)**  
: Node를 비우는 작업 (새 Pod는 못 오게 하고 기존 Pod를 다른 Node로 옮김)

**롤링 업그레이드(Rolling Upgrade)**  
: Pod를 하나씩 교체하며 무중단 업데이트하는 방식

**내결함성(Fault Tolerance)**  
: 일부 장애가 발생해도 시스템이 계속 동작하는 성질

**서비스 품질(QoS, Quality of Service)**  
: Kubernetes가 Pod 중요도를 나누는 등급 시스템으로, 리소스 부족 시 어떤 Pod를 먼저 죽일지 판단할 때 사용함

**노드 압박(Node Pressure)**  
: Node의 CPU·메모리·디스크 같은 리소스가 부족한 상태
