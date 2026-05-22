# Docs 스터디 커리큘럼 (2~7주차)

> 1주차: OT  
> 2~3주차: Terraform Docs  
> 4~7주차: Cilium Docs

---

## Terraform (2~3주차)

### 2주차 — 언어와 리소스
- [Configuration Language](https://developer.hashicorp.com/terraform/language) — 기본 문법, 파일 구조
  - Blocks, Arguments, Expressions
  - Types & Values, Strings & Templates
- [Providers](https://developer.hashicorp.com/terraform/language/providers)
- [Resources](https://developer.hashicorp.com/terraform/language/resources) — lifecycle, depends_on, count, for_each
- [Data Sources](https://developer.hashicorp.com/terraform/language/data-sources)

### 3주차 — 모듈과 상태 관리
- [Variables & Outputs](https://developer.hashicorp.com/terraform/language/values)
- [Modules](https://developer.hashicorp.com/terraform/language/modules) — 구조, 재사용, 레지스트리
- [State](https://developer.hashicorp.com/terraform/language/state) — 목적, 원격 상태, locking
- [Backends](https://developer.hashicorp.com/terraform/language/backend)
- [Workspaces](https://developer.hashicorp.com/terraform/language/state/workspaces)
- [CLI Commands](https://developer.hashicorp.com/terraform/cli/commands) — init, plan, apply, destroy, import, fmt, validate

---

## Cilium (4~7주차)

### 4주차 — 아키텍처와 네트워킹
- [Introduction & Architecture](https://docs.cilium.io/en/stable/overview/intro/)
- [Getting Started / Installation](https://docs.cilium.io/en/stable/gettingstarted/)
- [Networking](https://docs.cilium.io/en/stable/network/) — CNI, IPAM, IP 주소 관리
- [eBPF 기반 데이터 플레인](https://docs.cilium.io/en/stable/overview/component-overview/)

### 5주차 — 보안과 네트워크 정책
- [Network Policy](https://docs.cilium.io/en/stable/security/policy/) — L3/L4/L7 정책
- [Kubernetes Network Policy](https://docs.cilium.io/en/stable/security/policy/kubernetes/)
- [Host Firewall](https://docs.cilium.io/en/stable/security/host-firewall/)

### 6주차 — 가시성과 모니터링
- [Hubble — 가시성 및 모니터링](https://docs.cilium.io/en/stable/observability/hubble/)
  - Hubble CLI, Hubble UI
- [Metrics](https://docs.cilium.io/en/stable/observability/metrics/)
- [Policy Verdicts](https://docs.cilium.io/en/stable/observability/policy-verdicts/)

### 7주차 — 서비스 메시와 고급 기능
- [Service Mesh](https://docs.cilium.io/en/stable/network/servicemesh/)
- [Load Balancing](https://docs.cilium.io/en/stable/network/lb-ipam/)
- [Multi-cluster (Cluster Mesh)](https://docs.cilium.io/en/stable/network/clustermesh/)
- [Troubleshooting](https://docs.cilium.io/en/stable/operations/troubleshooting/)
