# Kubernetes Troubleshooting & Concepts

## 1. CoreDNS 부정 캐시 문제 (Negative DNS Cache)

### 증상
```
java.net.UnknownHostException: mysql-caas
Caused by: Temporary failure in name resolution
```
파드는 Running인데 서비스 이름으로 연결이 안 됨.

### 원인
파드 기동 시점에 CoreDNS가 서비스 이름을 못 찾으면 그 결과를 일정 시간 캐싱한다.
이후 서비스가 정상 등록되어도 캐시가 살아있는 동안 계속 실패.

### 해결
```bash
kubectl rollout restart deployment/coredns -n kube-system
```

### 예방
Deployment에 `initialDelaySeconds`를 충분히 주거나, DB가 완전히 뜬 뒤 앱을 배포.

### 참고
- [DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [Debugging DNS Resolution](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/)

---

## 2. StorageClass / PersistentVolume

### 증상
StatefulSet PVC가 Pending 상태에서 안 풀림.

### 원인
`storageClassName: standard`로 지정했는데 새 클러스터에 해당 StorageClass가 없음.

### 해결 (베어메탈 로컬 스토리지)
```bash
# local-path provisioner 설치
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml

# standard 이름으로 StorageClass 생성 (yaml 수정 없이)
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: rancher.io/local-path
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
EOF
```

### 참고
- [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [local-path-provisioner](https://github.com/rancher/local-path-provisioner)

---

## 3. nginx Ingress Controller - 베어메탈 외부 트래픽

### 증상
`LoadBalancer` 타입 서비스의 `EXTERNAL-IP`가 `<pending>`으로 유지됨.

### 원인
클라우드 환경이 아니면 LoadBalancer를 자동 할당해주는 컨트롤러가 없음.

### 해결 - hostNetwork 방식 (가정집 단일 공인 IP)
nginx ingress controller를 호스트 네트워크에 직접 바인딩.
포트 80/443을 NodePort 거치지 않고 호스트 포트로 직접 수신.

`ingress-nginx.yaml` Deployment spec에 추가:
```yaml
spec:
  template:
    spec:
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      nodeSelector:
        kubernetes.io/hostname: k8s-master  # 공유기 포트포워딩 대상 노드
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
```

> `dnsPolicy: ClusterFirstWithHostNet` — hostNetwork 사용 시 클러스터 DNS 유지에 필수

### 공유기 설정
- 외부 80 → 192.168.0.101:80 포트포워딩
- DNS A레코드 → 공인 IP

### 참고
- [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [ingress-nginx bare-metal guide](https://kubernetes.github.io/ingress-nginx/deploy/baremetal/)

---

## 4. RBAC - ServiceAccount, Role, RoleBinding

### 개념
CaaS처럼 k8s API를 직접 호출하는 서비스는 전용 ServiceAccount와 권한이 필요.

```
ServiceAccount (server ns)
    ↓ RoleBinding
Role (default ns) → Deployment/Service/Ingress/Pod 권한
```

- `Role` — 특정 네임스페이스 내 리소스 권한
- `ClusterRole` — 클러스터 전체 권한
- `RoleBinding` — ServiceAccount에 Role 연결

### 참고
- [RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Service Accounts](https://kubernetes.io/docs/concepts/security/service-accounts/)

---

## 5. StatefulSet vs Deployment

| | Deployment | StatefulSet |
|---|---|---|
| 파드 이름 | 랜덤 suffix | 순서 있는 고정 이름 (mysql-0, mysql-1) |
| 스토리지 | 공유 가능 | 파드마다 독립 PVC 자동 생성 |
| 용도 | 무상태 앱 | DB처럼 상태 있는 앱 |

StatefulSet의 `volumeClaimTemplates`가 자동으로 `data-mysql-caas-0` 같은 PVC를 생성함.

### 참고
- [StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)

---

## 6. Ingress 라우팅 - rewrite-target

### 서비스별 개별 Ingress (caas.fast-cloud.kro.kr)
`path: /`에 `rewrite-target`이 필요 없음. 경로 그대로 전달.

### API Gateway Ingress (path prefix 방식)
`/caas/api/foo` → 백엔드에는 `/api/foo`만 전달하려면:
```yaml
annotations:
  nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
  - http:
      paths:
      - path: /caas(/|$)(.*)
        pathType: ImplementationSpecific
```

### 참고
- [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [nginx rewrite](https://kubernetes.github.io/ingress-nginx/examples/rewrite/)

---

## 7. YAML 주의사항

- 한글 등 멀티바이트 문자가 YAML에 있으면 일부 파서에서 `control characters are not allowed` 에러 발생
- 주석도 예외 아님 → 영문 주석 사용 권장
