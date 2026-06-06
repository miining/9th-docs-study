# Kubernetes Docs 스터디 정리 — 클러스터 운영 / 네트워킹

## 1. kubeadm 인증서 관리

### 핵심 스크랩

kubeadm으로 생성된 클라이언트 인증서는 **1년 후 만료**된다. 기본적으로 kubeadm은 클러스터 실행에 필요한 모든 인증서를 생성하지만, 사용자는 자체 인증서를 제공해 이 동작을 덮어쓸 수 있다.

- `--cert-dir` 플래그 또는 `ClusterConfiguration.certificatesDir` 필드에 지정된 디렉터리에 배치한다. 기본값은 `/etc/kubernetes/pki`.
- 갱신 명령은 `/etc/kubernetes/pki`에 저장된 CA(또는 프론트 프록시 CA) 인증서·키를 사용해 수행된다.
- 갱신 후 **컨트롤 플레인 파드를 재시작**해야 한다. 현재 일부 구성 요소는 인증서 동적 리로드를 지원하지 않기 때문.
- `kubeadm upgrade` 명령을 사용하면 컨트롤 플레인 인증서가 1년 단위로 **자동 갱신**된다.
- **HA 클러스터**라면 모든 컨트롤 플레인 노드에서 명령을 실행해야 한다.

### 스태틱 파드 재시작 절차 (스크랩)

스태틱 파드는 API 서버가 아닌 **로컬 kubelet이 관리**하므로 `kubectl delete`로 재시작할 수 없다.

1. `/etc/kubernetes/manifests/`에서 매니페스트 파일을 임시로 제거한다.
2. 20초 대기 (`KubeletConfiguration.fileCheckFrequency` 값 참고). 파드가 매니페스트 디렉터리에 없으면 kubelet이 파드를 종료한다.
3. 파일을 다시 원위치로 이동한다.
4. 또 한 번의 `fileCheckFrequency` 주기가 지나면 kubelet이 파드를 재생성하고 인증서 갱신이 완료된다.

### 보강: 선행지식

**PKI(Public Key Infrastructure)와 mTLS**
Kubernetes의 모든 컴포넌트 간 통신(kubelet ↔ API server, etcd ↔ API server 등)은 **mTLS(mutual TLS)** 로 이뤄진다. 즉 클라이언트와 서버가 서로의 인증서를 검증한다. 그래서 인증서가 단순히 "서버용" 하나가 아니라, API server·kubelet·controller-manager·scheduler·etcd 각각이 클라이언트/서버 인증서를 갖는다. `/etc/kubernetes/pki`를 `ls` 해보면 `apiserver.crt`, `apiserver-kubelet-client.crt`, `front-proxy-client.crt` 등 여러 쌍이 보이는 이유다.

**왜 CA로 "갱신"이 가능한가**
인증서 갱신(renew)은 새 키쌍과 CSR을 만든 뒤, 기존 **CA 개인키로 다시 서명**하는 것이다. CA 인증서 자체(`ca.crt`)는 보통 10년짜리라 자주 안 바뀌지만, 그 CA가 서명해주는 leaf 인증서들은 1년이다. 그래서 `ca.key`만 있으면 leaf 인증서를 얼마든지 재발급할 수 있다 — 갱신 명령이 `/etc/kubernetes/pki`의 CA 키를 사용하는 이유.

**왜 동적 리로드가 안 되는가 → 스태틱 파드 개념**
API server 같은 컨트롤 플레인 컴포넌트는 시작 시점에 인증서 파일을 읽어 메모리에 올린다. 파일이 디스크에서 바뀌어도 프로세스가 이를 다시 읽지 않으면 옛 인증서를 계속 쓴다. 그래서 재시작이 필요하다.

> **스태틱 파드(static pod)란?**
> 일반 파드는 사용자가 API server에 요청 → scheduler가 노드 배정 → kubelet이 실행한다. 반면 스태틱 파드는 **kubelet이 특정 디렉터리(`/etc/kubernetes/manifests/`)를 직접 감시**하다가, 매니페스트 파일이 있으면 그대로 실행한다. API server를 거치지 않는다. 컨트롤 플레인(api-server, etcd, controller-manager, scheduler) 자체가 스태틱 파드로 떠 있다 — **닭과 달걀 문제**(API server를 띄우려면 API server가 필요?)를 푸는 부트스트랩 방식이다.
> kubelet은 스태틱 파드에 대해 API server에 "미러 파드(mirror pod)"를 만들어주므로 `kubectl get pod`에는 보이지만, 삭제·수정은 매니페스트 파일을 통해서만 가능하다.

**`fileCheckFrequency` 디테일**
kubelet이 매니페스트 디렉터리를 폴링하는 주기(기본 20초). 문서가 "제거 후 20초 대기"를 시키는 건 이 폴링 한 사이클을 보장하기 위함이다. 폴링 기반이라 즉각 반응이 아니다.

---

## 2. API에 프로그래밍 방식으로 접근

### 핵심 스크랩

공식 클라이언트 라이브러리: **Go, Python, Java, dotnet, JavaScript, Haskell**. 그 외 커뮤니티 유지보수 라이브러리도 존재한다.

**Go 클라이언트(client-go)**

```bash
go get k8s.io/client-go@kubernetes-<kubernetes-version-number>
```

> `client-go`는 자체 API 오브젝트를 정의하므로, API 정의를 가져올 때 기본 리포지터리가 아닌 client-go에서 import 한다. 예: `import "k8s.io/client-go/kubernetes"` 가 맞다.

```go
package main

import (
  "context"
  "fmt"
  "k8s.io/apimachinery/pkg/apis/meta/v1"
  "k8s.io/client-go/kubernetes"
  "k8s.io/client-go/tools/clientcmd"
)

func main() {
  // kubeconfig에서 현재 콘텍스트 사용 (예: /root/.kube/config)
  config, _ := clientcmd.BuildConfigFromFlags("", "<path-to-kubeconfig>")
  clientset, _ := kubernetes.NewForConfig(config)
  pods, _ := clientset.CoreV1().Pods("").List(context.TODO(), v1.ListOptions{})
  fmt.Printf("There are %d pods in the cluster\n", len(pods.Items))
}
```

Go 클라이언트는 kubectl과 **동일한 kubeconfig**를 사용한다. 애플리케이션이 클러스터 내부 파드로 배치된 경우엔 "파드 내에서 API 접근" 방식(ServiceAccount 토큰)을 따른다.

### 보강: CS / K8s 개념

**kubeconfig 구조**
`~/.kube/config`는 세 가지 블록으로 구성된다: `clusters`(API server 주소 + CA), `users`(인증 정보: 인증서/토큰), `contexts`(cluster + user + namespace 조합). `BuildConfigFromFlags`는 이 파일을 파싱해 어느 클러스터에 어떤 신원으로 붙을지 결정한다.

**클러스터 외부 vs 내부 인증 차이 — 중요**
- **외부(개발 PC)**: kubeconfig의 클라이언트 인증서나 토큰으로 인증. 위 예제가 이 경우.
- **내부(파드 안)**: kubeconfig가 없다. 대신 ServiceAccount 토큰이 `/var/run/secrets/kubernetes.io/serviceaccount/`에 자동 마운트된다. 이때는 `rest.InClusterConfig()`를 쓴다. RBAC로 그 ServiceAccount에 권한을 줘야 동작한다. Operator/Controller 개발의 기본이다.

**왜 client-go가 자체 API 오브젝트를 정의하는가**
Go는 정적 타입 언어라 `Pod`, `Deployment` 같은 리소스를 strongly-typed struct로 다룬다. `clientset.CoreV1().Pods(ns).List(...)`처럼 그룹(Core)·버전(V1)·리소스(Pods)가 메서드 체인으로 드러나는 건 K8s API의 **GVK(Group/Version/Kind)** 구조를 그대로 코드로 옮긴 것이다.

> **이어지는 개념 흐름: client-go → informer → controller**
> 실무에서 단순 List보다 중요한 건 **informer**다. 매번 API server에 List를 날리면 부하가 크다. Informer는 한 번 List한 뒤 **Watch**로 변경 이벤트만 받아 로컬 캐시(indexer)를 갱신한다. 이 watch-and-cache 패턴이 모든 컨트롤러의 심장이다. (List-Watch → Informer → WorkQueue → Reconcile 루프로 이어진다.)

---

## 3. 다중 스케줄러 설정

### 핵심 스크랩

기본 스케줄러가 요구를 못 채우면 직접 스케줄러를 구현하거나, **기본 스케줄러와 여러 스케줄러를 동시에 운영**할 수 있다. 파드별로 어떤 스케줄러를 쓸지 지정 가능하다.

**스케줄러 패키징** — 바이너리를 컨테이너 이미지로 만든다.

```bash
git clone https://github.com/kubernetes/kubernetes.git
cd kubernetes
make
```

```dockerfile
FROM busybox
ADD ./_output/local/bin/linux/amd64/kube-scheduler /usr/local/bin/kube-scheduler
```

```bash
docker build -t gcr.io/my-gcp-project/my-kube-scheduler:1.0 .
gcloud docker -- push gcr.io/my-gcp-project/my-kube-scheduler:1.0
```

**두 번째 스케줄러를 두는 이유 (스크랩 비유)**

1. **설정을 완전히 다르게 부여하기 위해** — 같은 능력의 비서 둘에게 업무 지침서(설정 파일)를 다르게 준다. 기본 스케줄러는 일반 파드를 보편 규칙으로 배치하고, 두 번째 스케줄러(`my-scheduler`)는 "GPU 노드에만 배치"처럼 다른 설정값으로 실행한다.
2. **특정 파드만 전담시키기 위해** — 파드 YAML에 `schedulerName: my-scheduler` 한 줄을 넣으면, 기본 스케줄러는 "내 담당 아님" 하고 무시하고 두 번째 스케줄러가 낚아채 배치한다. 완벽한 역할 분담.

### 보강: 스케줄링 내부 개념

**스케줄러가 실제로 하는 일 — 2단계 파이프라인**
1. **Filtering(Predicates)**: 파드를 도저히 못 올리는 노드를 거른다. (자원 부족, 노드 셀렉터 불일치, taint 등)
2. **Scoring(Priorities)**: 남은 노드에 점수를 매겨 최적 노드 1개를 고른다. (자원 여유, 분산 정도 등)
선택 후 **Binding** — 파드의 `spec.nodeName`을 채워 API server에 기록한다. 실제 컨테이너 실행은 해당 노드 kubelet이 한다. **스케줄러는 "결정"만 하고 "실행"은 kubelet이 한다**는 점이 핵심.

**`schedulerName`이 동작하는 메커니즘**
모든 스케줄러는 `spec.nodeName`이 비어 있고 **자기 이름과 `spec.schedulerName`이 일치하는** 파드만 watch한다. 이름이 다르면 그냥 무시한다. 그래서 별도 락이나 조정 없이도 여러 스케줄러가 충돌 없이 공존한다 — 분산 시스템에서 흔한 "관심사 필터링으로 경합 회피" 패턴.

> **선행지식: 왜 직접 만들지 않고 Scheduler Framework / Scheduler Plugin을 쓰나**
> 위 문서는 "바이너리를 통째로 빌드"하는 옛 방식이다. 요즘은 **Scheduling Framework**가 표준이다. `PreFilter`, `Filter`, `Score`, `Bind` 등 확장점(extension point)에 플러그인만 끼워 넣어 기본 스케줄러를 커스터마이즈한다. 통째로 새 스케줄러를 짜는 것보다 안전하고 유지보수가 쉽다.

> **분산 시스템 관점: split brain 위험**
> 두 스케줄러가 같은 노드의 남은 자원을 동시에 노려 각자 다른 파드를 배치하면 자원 초과(over-commit)가 날 수 있다. 그래서 다중 스케줄러는 보통 **서로 겹치지 않는 파드 집합**(`schedulerName`으로 명확히 분리)을 담당하도록 설계한다. 같은 파드를 두 스케줄러가 노리면 race condition이 생긴다.

---

## 4. kubectl proxy — API 서버 접근

### 핵심 스크랩

```bash
kubectl proxy --port=8080
```

프록시 실행 중 curl/wget/브라우저로 API를 탐색할 수 있다.

```bash
curl http://localhost:8080/api/
```

```json
{
  "kind": "APIVersions",
  "versions": ["v1"],
  "serverAddressByClientCIDRs": [
    { "clientCIDR": "0.0.0.0/0", "serverAddress": "10.0.2.15:8443" }
  ]
}
```

**이 기능이 있는 이유 — "보안 인증을 대신 해주는 터널"**
API server는 HTTPS(TLS) + 토큰/인증서 기반 권한 인증을 반드시 거쳐야 한다. proxy 없이 직접 접근하면:

```bash
curl --cacert /path/to/ca.crt --header "Authorization: Bearer $TOKEN" https://<API-서버-주소>/api/v1/...
```

`kubectl proxy`는 `~/.kube/config`의 인증 정보를 읽어 복잡한 보안 통신을 백그라운드에서 처리한다. 사용자는 `localhost:8080`으로 **인증서·토큰 없이 단순 HTTP 요청**만 던지면 된다.

**주요 사용처 (스크랩)**
1. **쿠버네티스 대시보드 안전 접속** — 대시보드를 외부망(LoadBalancer/Ingress)에 노출하지 않고 `localhost`로만 본다.
2. **쉘 스크립트 자동화** — kubectl 텍스트 결과 파싱보다, 프록시 + curl로 순수 JSON을 받는 게 가공·처리에 유리.
3. **Operator/CRD 개발** — Raw JSON 구조를 직접 확인하며 Postman/브라우저로 디버깅.

**`kubectl proxy` vs `kubectl port-forward` (스크랩)**
- `kubectl proxy`: K8s **API server**와 통신하는 길을 뚫음 (클러스터 관리 목적).
- `kubectl port-forward`: API server가 아닌, 클러스터 내 **특정 파드/서비스**(웹서버, DB)에 내 PC를 직결 (앱 테스트 목적).

> 정리: K8s 자체 데이터·시스템 제어/조회 → `proxy`, 내가 띄운 MySQL/웹앱 접속 → `port-forward`.

### 보강: 개념

**인증(Authentication) vs 인가(Authorization)**
proxy가 대신 해주는 건 **인증**(너 누구냐 — 인증서/토큰)이다. 그 뒤 API server는 **인가**(RBAC: 너 이거 할 권한 있냐)를 별도로 검사한다. proxy를 켰다고 권한이 생기는 건 아니다 — kubeconfig에 담긴 신원의 RBAC 권한만큼만 접근된다.

**API 경로 구조: `/api` vs `/apis`**
- `/api/v1` : core(legacy) 그룹. Pod, Service, Node 등.
- `/apis/<group>/<version>` : 그 외 모든 그룹. 예) `/apis/apps/v1/deployments`, `/apis/batch/v1/jobs`.
스크랩의 `curl http://localhost:8080/api/`가 `versions: ["v1"]`만 반환하는 건 core 그룹이라 그렇다.

**`serverAddressByClientCIDRs`의 의미**
클라이언트 IP(CIDR)에 따라 어느 API server 주소로 가야 하는지 알려주는 필드. 멀티 네트워크/HA 환경에서 클라이언트마다 적절한 엔드포인트를 안내하기 위한 것. `0.0.0.0/0`은 "모든 클라이언트는 이 주소로"라는 기본값.

> **port-forward의 내부 동작**
> port-forward도 사실 API server를 거친다. 로컬 포트 → API server → kubelet → 파드로 SPDY/WebSocket 스트림을 터널링한다. 즉 "API server를 안 거친다"기보다 **목적지가 파드냐 API server 자신이냐**의 차이로 이해하는 게 정확하다.

**Konnectivity 서비스 (스크랩)**
Konnectivity 서비스는 컨트롤 플레인에 클러스터 통신을 위한 **TCP 수준 프록시**를 제공한다.
> 보강: 컨트롤 플레인(API server)이 노드/파드 네트워크에 접근해야 할 때(예: `kubectl logs`, `exec`, webhook 호출), 둘이 다른 네트워크에 있으면 직접 통신이 어렵다. Konnectivity는 노드 쪽 agent와 컨트롤 플레인 쪽 server가 **영구 터널**을 미리 맺어두고, 그 터널로 트래픽을 흘려보내 이 문제를 푼다. 과거 SSH 터널 방식을 대체한 것.

---

## 5. HostAliases — 파드의 /etc/hosts 항목 추가

### 핵심 스크랩

파드 `/etc/hosts`에 항목 추가는 DNS 등 다른 방법이 안 통할 때 **파드 수준 호스트네임 해석**을 제공한다. PodSpec의 `HostAliases`로 추가한다.

> HostAliases를 쓰지 않은 수정은 비권장. hosts 파일은 kubelet이 관리하며 파드 생성/재시작 중 **덮어쓰일 수 있다**.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostaliases-pod
spec:
  restartPolicy: Never
  hostAliases:
  - ip: "127.0.0.1"
    hostnames:
    - "foo.local"
    - "bar.local"
  - ip: "10.1.2.3"
    hostnames:
    - "foo.remote"
    - "bar.remote"
  containers:
  - name: cat-hosts
    image: busybox:1.28
    command:
    - cat
    args:
    - "/etc/hosts"
```

```bash
kubectl apply -f https://k8s.io/examples/service/networking/hostaliases-pod.yaml
kubectl get pod --output=wide   # IP, NODE 확인
kubectl logs hostaliases-pod    # hosts 내용 확인
```

결과 hosts 파일 끝부분:

```text
# Kubernetes-managed hosts file.
127.0.0.1	localhost
...
10.200.0.5	hostaliases-pod

# Entries added by HostAliases.
127.0.0.1	foo.local	bar.local
10.1.2.3	foo.remote	bar.remote
```

**왜 Kubelet이 hosts 파일을 관리하는가 — 중요 (스크랩)**
컨테이너 시작 후 컨테이너 런타임이 hosts를 멋대로 수정하는 것을 막기 위해, kubelet이 각 컨테이너의 hosts를 관리한다. 과거 Docker 엔진은 컨테이너 시작 후 `/etc/hosts`를 수정할 수 있었다. 현재는 런타임이 다양하지만, kubelet이 관리하므로 **어떤 런타임을 쓰든 동일한 결과**를 보장한다.

> **주의**: 컨테이너 내부 hosts를 수동 변경하면 안 된다. 컨테이너 종료 시 변경이 손실된다.

### 보강: 개념

**이름 해석 우선순위 — 왜 hosts가 DNS보다 먼저인가**
리눅스는 `/etc/nsswitch.conf`의 `hosts:` 줄(보통 `files dns`)에 따라 **`/etc/hosts`(files)를 먼저** 보고 없으면 DNS로 간다. 그래서 HostAliases로 박은 항목은 CoreDNS보다 우선한다. DNS를 우회한 "강제 매핑"이 가능한 이유.

**언제 HostAliases가 적절한가 / 부적절한가**
- 적절: DNS에 없는 외부 레거시 호스트명을 고정 IP로 강제 매핑, 테스트용 임시 오버라이드.
- 부적절: 클러스터 내부 서비스 디스커버리. 이건 **Service + CoreDNS**(`my-svc.my-ns.svc.cluster.local`)로 해야 한다. IP를 박아두면 파드 재스케줄로 IP가 바뀌었을 때 깨진다.

> **이어지는 개념: hostNetwork / dnsPolicy와의 관계**
> HostAliases는 hosts 파일만 건드린다. 더 넓게는 파드의 DNS 동작을 `dnsPolicy`(`ClusterFirst`, `Default`, `None`)와 `dnsConfig`로 제어한다. `hostNetwork: true`인 파드는 노드의 네트워크 네임스페이스를 그대로 쓰므로 DNS 정책 기본값도 달라진다(이때 `ClusterFirstWithHostNet` 필요). HostAliases → dnsPolicy → dnsConfig 순으로 "파드의 이름 해석을 어떻게 통제하나"를 정리해두면 좋다.

**왜 "kubelet이 관리"가 멱등성(idempotency) 보장인가**
컨테이너 런타임마다 hosts 처리 방식이 다르면 같은 매니페스트가 노드/런타임에 따라 다른 결과를 낸다. kubelet이 단일 책임으로 hosts를 생성하면 **선언적 구성(declarative config)** 의 핵심인 "같은 입력 → 같은 결과"가 지켜진다. 수동 수정이 손실되는 것도 같은 이유 — kubelet이 재생성 시 자기가 만든 버전으로 덮어쓴다.

---

## 부록: 이번 문서 전체를 관통하는 한 가지

다섯 주제가 따로 노는 것 같지만 공통 축은 **"kubelet과 API server의 역할 분리"** 다.

- 인증서: API server는 갱신된 인증서를 재시작해야 읽지만, 그 재시작을 거는 건 kubelet(스태틱 파드).
- client-go: API server에 붙는 외부 접근 vs 파드 내부 ServiceAccount 접근.
- 스케줄러: 배치 "결정"은 스케줄러, 실제 "실행"은 kubelet.
- proxy: 인증을 대신 처리해 API server에 닿게 해줌.
- HostAliases: 파드 hosts의 최종 관리 주체는 kubelet.

> **결론 한 줄**: K8s에서 "선언(API server에 기록)"과 "실행(kubelet이 노드에서 수행)"은 분리되어 있고, 거의 모든 운영 동작은 이 두 주체 중 누가 책임지는지를 따라가면 이해된다.