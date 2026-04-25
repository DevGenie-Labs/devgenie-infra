# devgenie-infra CLAUDE.md

## 이 레포가 하는 일

DevGenie의 **환경 설정 + 운영 인프라** 레포다.
Terraform이 VM/K8s 리소스를 **만든 후**, 이 레포의 Ansible이 **내용을 채운다**.

담당 범위:
- Ansible 플레이북 — code-server 설치, Runtime 환경 구성
- Kubernetes 매니페스트 — Ingress Controller, Namespace 정책
- Nginx 설정 — Reverse Proxy, SSL 종료
- pfSense/WireGuard 설정 문서 및 스크립트

**실행 주체는 Worker VM이다.**

---

## 기술 스택

- Ansible (플레이북 기반 환경 설정)
- Kubernetes YAML (매니페스트)
- Nginx (Reverse Proxy 설정)
- WireGuard (VPN 설정)
- Shell Script

---

## 디렉토리 구조

```
devgenie-infra/
├── ansible/
│   ├── playbooks/
│   │   ├── workspace-setup.yml     # code-server + Runtime 설치 (Workspace 생성 시 실행)
│   │   ├── workspace-teardown.yml  # 환경 정리 (삭제 시)
│   │   └── base-setup.yml         # VM 초기 세팅 (1회성)
│   ├── roles/
│   │   ├── code-server/            # code-server 설치/설정
│   │   ├── java/                   # Java Runtime
│   │   ├── node/                   # Node.js Runtime
│   │   └── python/                 # Python Runtime
│   ├── inventory/
│   │   ├── hosts.ini               # (커밋 금지 — gitignore)
│   │   └── hosts.ini.example
│   └── group_vars/
│       └── all.yml
├── k8s/
│   ├── ingress-controller/         # Nginx Ingress Controller 배포
│   ├── namespace-policy/           # NetworkPolicy, ResourceQuota
│   └── monitoring/                 # Phase 2: Prometheus, Grafana
├── nginx/
│   ├── nginx.conf                  # 메인 설정
│   └── conf.d/
│       └── devgenie.conf           # devgenie.io 도메인 설정
├── scripts/
│   └── health-check.sh
└── docs/
    └── network-topology.md
```

---

## Ansible 실행 규칙

### Workspace 생성 시 (Worker가 호출)
```bash
ansible-playbook playbooks/workspace-setup.yml \
  -i inventory/hosts.ini \
  --extra-vars "workspace_id=ws-abc123 runtime=java17 namespace=ws-abc123"
```

### 중요 규칙
- `inventory/hosts.ini`는 절대 커밋하지 않는다 (SSH 키 포함 위험)
- `hosts.ini.example`만 커밋한다
- 플레이북은 멱등성 보장 (여러 번 실행해도 동일 결과)
- 민감 정보(SSH 비밀번호, API 키)는 Ansible Vault로 암호화

---

## 네트워크 구성 참고

```
External (DMZ): 192.168.1.0/24  → HTTPS 443, WireGuard 51820
Private:        10.0.1.0/24     → 내부 서비스 간 통신
K8s Pod:        10.244.0.0/16   → Pod 간 통신
Management:     10.10.0.0/24    → WireGuard VPN (관리자만)
```

VM 구성:
```
VM1: Proxy/Bastion  (2 vCPU / 4GB)   — DMZ
VM2: Spring Boot    (4 vCPU / 8GB)   — Private
VM3: Worker + AI    (8 vCPU / 16GB)  — Private
VM4: PostgreSQL     (4 vCPU / 8GB)   — Private
VM5: K8s Master     (4 vCPU / 8GB)   — Private
VM6: K8s Worker(s)  (8+ vCPU / 16GB+)— Private
```

---

## 코딩 규칙

- Ansible 변수명: snake_case (`workspace_id`, `runtime_version`)
- 플레이북 파일명: kebab-case (`workspace-setup.yml`)
- K8s 매니페스트: 리소스 타입별 파일 분리
- 커밋 금지 파일: `hosts.ini`, `*.pem`, `*.key`, `vault_password`

---

## 연관 레포

- `devgenie-terraform` — Terraform이 리소스를 먼저 만들고, Ansible이 그 위에서 실행됨
- `devgenie-ai-engine` — Worker: 이 Ansible 플레이북을 CLI로 실행하는 주체
