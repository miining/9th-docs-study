## Service / Storage / Ingress에서 처음 본 개념들 (2주차로 미룸)

## **IPv4/IPv6 이중 스택**

클러스터에서 IPv4와 IPv6 주소를 동시에 사용하는 설정.

## **Topology Aware Routing**

쿠버네티스 멀티 존(Multi-zone) 환경에서 **네트워크 트래픽이 발생한 동일한 존 내부의 포드(Pod)로 트래픽을 우선 라우팅**해주는 기능입니다. 

- **EndpointSlice 컨트롤러:** 각 존에 있는 노드들의 **할당 가능한(Allocatable) CPU 코어 수**를 기준으로, 엔드포인트(포드)를 각 존에 비례하게 할당하고 `hints` 필드를 채웁니다. (예: CPU가 많은 존에 더 많은 엔드포인트 배치)
- **kube-proxy:** EndpointSlice에 설정된 `hints`를 읽어 들여, 가능한 한 동일한 존 내부의 엔드포인트로 트래픽을 필터링하여 보냅니다.

## **토폴로지 인지 힌트**

쿠버네티스 클러스터가 멀티-존(multi-zone) 환경에 배포되는 일이 점점 많아지고 있다.

`service.kubernetes.io/topology-aware-hints` 어노테이션을 `auto`로 설정하여 서비스에 대한 토폴로지 인지 힌트를 활성화

## **서비스 내부 트래픽 정책**

[서비스](https://kubernetes.io/ko/docs/concepts/services-networking/service/)의 `.spec.internalTrafficPolicy`를 `Local`로 설정하여 내부 전용 트래픽 정책을 활성화 할 수 있다. 이것은 kube-proxy가 클러스터 내부 트래픽을 위해 노드 내부 엔드포인트로만 사용하도록 한다.

**지정된 서비스에 대한 엔드포인트가 없는 노드의 파드인 경우에 서비스는 다른 노드에 엔드포인트가 있더라도 엔드포인트가 없는 것처럼 작동한다. (이 노드의 파드에 대해서)**

 노드의 캐시가 죽자마자 패킷이 즉시 드랍됩니다. 연결이 뚝 끊기니 마크 앱이 **"어? 내 컴퓨터에 있는 캐시 죽었네? 위험하다!"** 하고 즉시 눈치를 채고 백업 시스템을 가동하거나 에러 로그를 뿜습니다.

## 볼륨

도커 볼륨은 디스크에 있는 디렉터리이거나 다른 컨테이너에 있다. 도커는 볼륨 드라이버를 제공하지만, 기능이 다소 제한된다.

기본적으로 볼륨은 디렉터리이며, 일부 데이터가 있을 수 있으며, 파드 내 컨테이너에서 접근할 수 있다. 

## **클라우드 네이티브 보안**

![](https://kubernetes.io/images/docs/4c.png)

클라우드 네이티브 보안의 4C는 클라우드(Cloud), 클러스터(Cluster), 컨테이너(Container)와 코드(Code)이다.

파드 시큐리티 스탠다드에서는 보안 범위를 넓게 다루기 위해 세 가지 *정책을* 정의한다. 이러한 정책은 *점증적이며* 매우 허용적인 것부터 매우 제한적인 것까지 있다. 이 가이드는 각 정책의 요구사항을 간략히 설명한다.

| 프로필 | 설명 |
| --- | --- |
| **특권(Privileged)** | 무제한 정책으로, 가장 넓은 범위의 권한 수준을 제공한다. 이 정책은 알려진 권한 상승(privilege escalations)을 허용한다. |
| **기본(Baseline)** | 알려진 권한 상승을 방지하는 최소한의 제한 정책이다. 기본(최소로 명시된) 파드 구성을 허용한다. |
| **제한(Restricted)** | 엄격히 제한된 정책으로 현재 파드 하드닝 모범 사례를 따른다. |

쿠버네티스 [파드 시큐리티 스탠다드](https://kubernetes.io/ko/docs/concepts/security/pod-security-standards/)는 파드에 대해 서로 다른 격리 수준을 정의한다. 이러한 표준을 사용하면 파드의 동작을 명확하고 일관된 방식으로 제한하는 방법을 정의할 수 있다.

쿠버네티스는 파드 시큐리티 스탠다드를 적용하기 위해 내장된 *파드 시큐리티* [어드미션 컨트롤러](https://kubernetes.io/ko/docs/reference/access-authn-authz/admission-controllers/)를 제공한다. 파드 시큐리티의 제한은 파드가 생성될 때 [네임스페이스](https://kubernetes.io/ko/docs/concepts/overview/working-with-objects/namespaces/) 수준에서 적용된다.

## **네임스페이스에 대한 파드 시큐리티 어드미션 레이블**

이 기능이 활성화되거나 웹훅이 설치되면, 네임스페이스를 구성하여 각 네임스페이스에서 파드 보안에 사용할 어드미션 제어 모드를 정의할 수 있다. 쿠버네티스는 미리 정의된 파드 시큐리티 스탠다드 수준을 사용자가 네임스페이스에 정의하여 사용할 수 있도록 [레이블](https://kubernetes.io/ko/docs/concepts/overview/working-with-objects/labels) 집합을 정의한다. 선택한 레이블은 잠재적인 위반이 감지될 경우 [컨트롤 플레인](https://kubernetes.io/ko/docs/reference/glossary/?all=true#term-control-plane)이 취하는 조치를 정의한다.

| 모드 | 설명 |
| --- | --- |
| **강제(enforce)** | 정책 위반 시 파드가 거부된다. |
| **감사(audit)** | 정책 위반이 [감사 로그](https://kubernetes.io/ko/docs/tasks/debug/debug-cluster/audit/)에 감사 어노테이션 이벤트로 추가되지만, 허용은 된다. |
| **경고(warn)** | 정책 위반이 사용자에게 드러나도록 경고를 트리거하지만, 허용은 된다. |

## **서비스 어카운트란 무엇인가?**

서비스 어카운트란 사람이 사용하지 않는 계정으로, 쿠버네티스 클러스터 내에서 구분되는 신원을 제공한다. 애플리케이션 파드, 시스템 컴포넌트, 그리고 클러스터 내부와 외부의 엔티티들은 특정 서비스어카운트(ServiceAccount)의 자격 증명을 사용하여 해당 서비스어카운트(ServiceAccount)로 식별될 수 있다. 이러한 신원은 API 서버에 대한 인증이나 신원 기반의 보안 정책을 구현하는 등 다양한 상황에서 유용하다.

| 설명 | 서비스어카운트(ServiceAccount) | 유저 혹은 그룹 |
| --- | --- | --- |
| 위치 | 쿠버네티스 API 서비스어카운트 오브젝트(ServiceAccount object) | 외부 |
| 접근 제어 | 쿠버네티스 RBAC 또는 기타 [인가 메커니즘](https://kubernetes.io/ko/docs/reference/access-authn-authz/authorization/#authorization-modules) | 쿠버네티스 RBAC 또는 기타 신원 및 접근 관리 메커니즘 |
| 사용 목적 | 워크로드, 자동화 | 사람 |

## **서비스어카운트(ServiceAccount)에 대한 권한 부여**

쿠버네티스에 내장된 [역할 기반 접근 제어(RBAC)](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) 매커니즘을 활용하면 각 서비스 어카운트에 필요한 최소한의 권한을 부여할 수 있다. 먼저 접근 권한을 정의하는 *role*을 생성하고, 이를 서비스어카운트(ServiceAccount)에 *바인딩*한다. RBAC을 사용하면 최소 권한 원칙에 따라 서비스 어카운트의 권한을 정의할 수 있다. 따라서 해당 서비스 어카운트를 사용하는 파드는 올바르게 동작하는 데 필요한 권한 이상을 가지지 않는다.

## **쿠버네티스 API 접근 제어하기**

![](https://kubernetes.io/images/docs/admin/access-control-overview.svg)

## **쿠버네티스 스케줄러**

스케줄러는 파드가 실행 가능한 노드를 찾은 다음 실행 가능한 노드의 점수를 측정하는 기능 셋을 수행하고 실행 가능한 노드 중에서 가장 높은 점수를 가진 노드를 선택하여 파드를 실행한다. 그런 다음 스케줄러는 *바인딩* 이라는 프로세스에서 이 결정에 대해 API 서버에 알린다.

kube-scheduler는 2단계 작업에서 파드에 대한 노드를 선택한다.

1. 필터링
2. 스코어링(scoring)

## **테인트(Taints)와 톨러레이션(Tolerations)**

[*노드 어피니티*](https://kubernetes.io/ko/docs/concepts/scheduling-eviction/assign-pod-node/#%EC%96%B4%ED%94%BC%EB%8B%88%ED%8B%B0-affinity-%EC%99%80-%EC%95%88%ED%8B%B0-%EC%96%B4%ED%94%BC%EB%8B%88%ED%8B%B0-anti-affinity)는 [노드](https://kubernetes.io/ko/docs/concepts/architecture/nodes/) 셋을 (기본 설정 또는 어려운 요구 사항으로) *끌어들이는* [파드](https://kubernetes.io/ko/docs/concepts/workloads/pods/)의 속성이다. *테인트* 는 그 반대로, 노드가 파드 셋을 제외시킬 수 있다.

*톨러레이션* 은 파드에 적용된다. 톨러레이션을 통해 스케줄러는 그와 일치하는 테인트가 있는 파드를 스케줄할 수 있다. 톨러레이션은 스케줄을 허용하지만 보장하지는 않는다. 스케줄러는 그 기능의 일부로서 [다른 매개변수를](https://kubernetes.io/ko/docs/concepts/scheduling-eviction/pod-priority-preemption/) 고려한다.

테인트와 톨러레이션은 함께 작동하여 파드가 부적절한 노드에 스케줄되지 않게 한다. 하나 이상의 테인트가 노드에 적용되는데, 이것은 노드가 테인트를 용인하지 않는 파드를 수용해서는 안 된다는 것을 나타낸다.

## **우선순위와 선점을 사용하는 방법**

우선순위와 선점을 사용하려면 다음을 참고한다.

1. 하나 이상의 [프라이어리티클래스](https://kubernetes.io/ko/docs/concepts/scheduling-eviction/pod-priority-preemption/#%ED%94%84%EB%9D%BC%EC%9D%B4%EC%96%B4%EB%A6%AC%ED%8B%B0%ED%81%B4%EB%9E%98%EC%8A%A4)를 추가한다.
2. 추가된 프라이어리티클래스 중 하나에 [`priorityClassName`](https://kubernetes.io/ko/docs/concepts/scheduling-eviction/pod-priority-preemption/#%ED%8C%8C%EB%93%9C-%EC%9A%B0%EC%84%A0%EC%88%9C%EC%9C%84)이 설정된 파드를 생성한다. 물론 파드를 직접 생성할 필요는 없다. 일반적으로 디플로이먼트와 같은 컬렉션 오브젝트의 파드 템플릿에 `priorityClassName` 을 추가한다.