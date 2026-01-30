# Avadot EKS Infrastructure

Avadot 메타휴먼 서비스의 Kubernetes(EKS) 마이그레이션을 위한 인프라 프로젝트입니다.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           Unreal Engine 5 Client                                │
│                          (MetaHuman + LiveLink)                                 │
└───────────────────────────────┬─────────────────────────────────────────────────┘
                                │
                ┌───────────────┴───────────────┐
                │                               │
                ▼                               ▼
        REST/WebSocket                        gRPC
        (대화, TTS 요청)                    (음성 → 얼굴 애니메이션)
                │                               │
┌───────────────┴───────────────────────────────┴───────────────────────────────┐
│                              AWS EKS Cluster                                   │
│                                                                                │
│  ┌────────────────────────────────┐    ┌────────────────────────────────────┐ │
│  │  Node Group: avadot-nodes      │    │  Node Group: avadot-node-gpu       │ │
│  │  (t3.large × 2)                │    │  (g4dn.xlarge × 1)                 │ │
│  │                                │    │                                    │ │
│  │  ┌──────────┐ ┌──────────┐    │    │  ┌────────────────────────────┐   │ │
│  │  │ backend  │ │  tarot   │    │    │  │    a2f-grpc-server         │   │ │
│  │  │  :4008   │ │  :4007   │    │    │  │    :50051 (gRPC)           │   │ │
│  │  └──────────┘ └──────────┘    │    │  │                            │   │ │
│  │  ┌──────────┐ ┌──────────┐    │    │  │  NVIDIA Audio2Face SDK     │   │ │
│  │  │custom_cha│ │blind_date│    │    │  │  - Diffusion Model         │   │ │
│  │  │  :4009   │ │  :4010   │    │    │  │  - TensorRT Engine         │   │ │
│  │  └──────────┘ └──────────┘    │    │  │  - 68 Blendshapes @30FPS   │   │ │
│  │  ┌──────────┐ ┌──────────┐    │    │  └────────────────────────────┘   │ │
│  │  │concierge │ │fact_worke│    │    │                                    │ │
│  │  │  :4011   │ │ (worker) │    │    │  GPU: T4 16GB                      │ │
│  │  └──────────┘ └──────────┘    │    │  CUDA 12.8+ / TensorRT 10.8+       │ │
│  └────────────────────────────────┘    └────────────────────────────────────┘ │
│                │                                      │                        │
│                └──────────────┬───────────────────────┘                        │
│                               │                                                │
│                    ┌──────────┴──────────┐                                     │
│                    ▼                     ▼                                     │
│               ┌────────┐           ┌────────┐                                  │
│               │  RDS   │           │ Redis  │                                  │
│               │ MySQL  │           │ Cache  │                                  │
│               └────────┘           └────────┘                                  │
└────────────────────────────────────────────────────────────────────────────────┘
```

## Project Structure

```
a2f-infra/
├── base/                       # K8s Base Manifests (Kustomize)
│   ├── backend/               # REST API 서비스
│   ├── tarot/                 # 타로 WebSocket 서비스
│   ├── custom_chat/           # 커스텀 챗 WebSocket 서비스
│   ├── blind_date/            # 소개팅 WebSocket 서비스
│   ├── concierge/             # 메타 챗봇 서비스
│   └── fact_worker/           # 백그라운드 워커
│
├── overlays/                   # Environment-specific Patches
│   ├── local/                 # 로컬 개발 환경 (NodePort)
│   └── prod/                  # 프로덕션 환경 (ALB Ingress)
│
├── avadot/                     # Backend Application (Git Submodule)
│   ├── services/              # FastAPI 마이크로서비스
│   ├── core/                  # 공유 모듈
│   └── docker-compose.yml     # 기존 Docker Compose 설정
│
├── a2f-grpc-server/            # GPU gRPC Server (Git Submodule)
│   ├── server/                # gRPC 서비스 구현
│   ├── executor/              # NVIDIA SDK 바인딩
│   └── docker/                # GPU Docker 설정
│
└── docs/                       # Documentation
```

## Node Groups

| Node Group | Instance Type | Count | Purpose | Services |
|------------|---------------|-------|---------|----------|
| `avadot-nodes` | t3.large | 2 | CPU 백엔드 | backend, tarot, custom_chat, blind_date, concierge, fact_worker |
| `avadot-node-gpu` | g4dn.xlarge | 1 | GPU 추론 | a2f-grpc-server |

## Services

### CPU Services (avadot-nodes)

| Service | Port | Description |
|---------|------|-------------|
| backend | 4008 | REST API (세션, 캐릭터, 사용자 관리) |
| tarot | 4007 | 타로 상담 WebSocket |
| custom_chat | 4009 | 커스텀 AI 챗 WebSocket |
| blind_date | 4010 | 소개팅 AI 챗 WebSocket |
| concierge | 4011 | Cross-service 메타 챗봇 (RAG) |
| fact_worker | - | 사실 추출 백그라운드 워커 |

### GPU Services (avadot-node-gpu)

| Service | Port | Description |
|---------|------|-------------|
| a2f-grpc-server | 50051 | NVIDIA Audio2Face gRPC 서버 |

## Data Flow

```
1. User Input (대화)
       │
       ▼
2. Avadot Backend (CPU Node)
   ├─ LLM 응답 생성 (Claude/OpenAI)
   └─ TTS 음성 생성 (ElevenLabs)
       │
       ▼
3. A2F gRPC Server (GPU Node)
   ├─ 음성 분석
   └─ 얼굴 애니메이션 생성
      - ARKit 52 + Tongue 16 = 68 Blendshapes
      - 30 FPS 출력
       │
       ▼
4. Unreal Engine Client
   ├─ LiveLink 플러그인으로 수신
   └─ MetaHuman에 실시간 적용
      - 립싱크
      - 표정 애니메이션
```

## Quick Start

### Prerequisites

- AWS CLI configured
- kubectl installed
- kustomize installed

### Deploy to Local (Minikube/Kind)

```bash
# Apply local overlay
kubectl apply -k overlays/local
```

### Deploy to Production

```bash
# Apply production overlay
kubectl apply -k overlays/prod
```

### Check Status

```bash
# View all pods
kubectl get pods -n avadot-prod -o wide

# View services
kubectl get svc -n avadot-prod

# View nodes
kubectl get nodes
```

## Resource Configuration

### Current Cluster Capacity

| Resource | Per Node (t3.large) | Total (2 nodes) |
|----------|---------------------|-----------------|
| CPU | 2 vCPU | 4 vCPU |
| Memory | 8 GB | 16 GB |

### Per-Pod Resources

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

### Scaling Configuration

| Config | Min | Max | Trigger |
|--------|-----|-----|---------|
| Node Group (CPU) | 2 | 2 | Fixed |
| HPA (per service) | 2 | 4 | CPU > 70% |

## Related Repositories

| Repository | Description |
|------------|-------------|
| [avadot](./avadot) | FastAPI 백엔드 서비스 |
| [a2f-grpc-server](./a2f-grpc-server) | NVIDIA Audio2Face gRPC 서버 |
| [Audio2Face-3D-SDK](https://github.com/tridot-io/Audio2Face-3D-SDK) | NVIDIA SDK Fork |

## License

Private - Tridot Inc.
