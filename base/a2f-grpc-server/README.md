# A2F gRPC Server - Kubernetes 배포 가이드

Audio2Face Diffusion 모델 기반 gRPC 서버의 Kubernetes 배포 가이드입니다.

## 목차

- [아키텍처 개요](#아키텍처-개요)
- [TensorRT 엔진 빌드](#tensorrt-엔진-빌드)
- [배포 전 준비사항](#배포-전-준비사항)
- [배포 방법](#배포-방법)
- [환경변수](#환경변수)
- [트러블슈팅](#트러블슈팅)

## 아키텍처 개요

```
┌─────────────────────────────────────────────────────────────────┐
│                         EKS Cluster                              │
│                                                                  │
│  ┌────────────────────┐    ┌──────────────────────────────────┐ │
│  │  Backend Pod       │    │  GPU Nodes (avadot-node-gpu × 2) │ │
│  │  (CPU Node)        │    │                                  │ │
│  │                    │    │  ┌────────────┐ ┌────────────┐   │ │
│  │  avadot-backend    │───▶│  │ server-0   │ │ server-1   │   │ │
│  │  :4008             │gRPC│  │ :50051     │ │ :50051     │   │ │
│  │                    │    │  │ PVC: 50Gi  │ │ PVC: 50Gi  │   │ │
│  └────────────────────┘    │  └────────────┘ └────────────┘   │ │
│                            │                                  │ │
│                            │  StatefulSet (replicas: 2)       │ │
│                            │  PDB: minAvailable 1             │ │
│                            └──────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

**핵심 구성**:
- **StatefulSet**: Pod별 독립 PVC (volumeClaimTemplates)
- **PDB**: minAvailable 1 → 노드 업데이트 시 1개 Pod 항상 유지
- **Service**: ClusterIP (Ingress용) + Headless (StatefulSet DNS용)

## TensorRT 엔진 빌드

### GPU 종속성 이해

⚠️ **중요**: TensorRT(TRT) 엔진 파일(`.trt`)은 GPU 아키텍처에 종속됩니다.

| GPU | Compute Capability | 비고 |
|-----|-------------------|------|
| T4 (Turing) | sm_75 | g4dn 인스턴스 |
| A10G (Ampere) | sm_86 | g5 인스턴스 |
| A100 (Ampere) | sm_80 | p4d 인스턴스 |

로컬에서 빌드한 `.trt` 파일은 다른 GPU 아키텍처에서 사용할 수 없습니다.

### 자동 TRT 빌드 (entrypoint.sh)

컨테이너 시작 시 `entrypoint.sh`가 자동으로 TRT 엔진을 빌드합니다:

1. **GPU 아키텍처 감지**: `nvidia-smi`로 현재 GPU의 compute capability 확인
2. **캐시 확인**: `.gpu_arch` 마커 파일로 기존 빌드와 비교
3. **조건부 빌드**:
   - TRT 파일이 없거나
   - GPU 아키텍처가 변경되었거나
   - ONNX 파일이 더 최신인 경우 → 재빌드
4. **빌드 실행**: `trtexec`로 ONNX → TRT 변환

### 빌드 시간

- 첫 배포 시: **5-10분** (TRT 엔진 빌드)
- 이후 배포: **즉시** (PVC에 캐시된 TRT 사용)

### PVC 요구사항

각 Pod이 독립 PVC를 가짐 (volumeClaimTemplates):
- `model-data-a2f-grpc-server-0` (50Gi, gp2)
- `model-data-a2f-grpc-server-1` (50Gi, gp2)

PVC에 저장되는 내용:
- 모델 데이터 (S3에서 initContainer가 다운로드)
- TRT 엔진 파일 (`network.trt`) - 자동 생성
- GPU 아키텍처 마커 (`.gpu_arch`) - 자동 생성

## 배포 전 준비사항

### 1. GPU 노드 그룹 생성

```bash
# eksctl로 GPU 노드 그룹 추가
eksctl create nodegroup \
  --cluster avadot-cluster \
  --name avadot-node-gpu \
  --node-type g4dn.xlarge \
  --nodes 2 \
  --nodes-min 2 \
  --nodes-max 2 \
  --node-labels "alpha.eksctl.io/nodegroup-name=avadot-node-gpu" \
  --install-nvidia-plugin
```

### 2. 모델 데이터

initContainer가 S3에서 자동으로 모델을 다운로드합니다:
- 소스: `s3://avadot/a2f-models/`
- 대상: `/data/models/` (PVC)
- 이미 존재하면 스킵

### 3. ECR 이미지 푸시

```bash
# ECR 로그인
aws ecr get-login-password --region ap-northeast-2 | \
  docker login --username AWS --password-stdin 247124030538.dkr.ecr.ap-northeast-2.amazonaws.com

# 이미지 빌드 및 푸시
docker build -f docker/Dockerfile -t a2f-grpc-server:latest .
docker tag a2f-grpc-server:latest 247124030538.dkr.ecr.ap-northeast-2.amazonaws.com/a2f-grpc-server:latest
docker push 247124030538.dkr.ecr.ap-northeast-2.amazonaws.com/a2f-grpc-server:latest
```

## 배포 방법

### Kustomize 배포

```bash
# Dry-run 검증
kubectl apply -k base/a2f-grpc-server --dry-run=client

# 실제 배포
kubectl apply -k base/a2f-grpc-server

# 배포 상태 확인 (StatefulSet은 순차 배포: Pod 0 Ready → Pod 1 생성)
kubectl get pods -l app=a2f-grpc-server -w

# Pod이 서로 다른 노드에 배포되었는지 확인
kubectl get pods -l app=a2f-grpc-server -o wide
```

### PDB 확인

```bash
kubectl get pdb a2f-grpc-server-pdb
# ALLOWED DISRUPTIONS: 1 이면 정상
```

### 로그 확인

```bash
# TRT 빌드 로그 (첫 배포 시)
kubectl logs a2f-grpc-server-0 -f
kubectl logs a2f-grpc-server-1 -f

# 로그에서 확인할 내용:
# [entrypoint] Initializing A2F Server...
# [entrypoint] Building TRT from ONNX: multi-diffusion
# [entrypoint] TRT engine created: /data/models/multi-diffusion/network.trt
# [entrypoint] Starting server...
```

## 환경변수

| 변수명 | 설명 | 기본값 |
|--------|------|--------|
| `A2F_MODEL_DATA_PATH` | 모델 데이터 디렉토리 | `/data/models` |
| `A2F_MODEL_PATH` | model.json 경로 | `/data/models/multi-diffusion/model.json` |
| `A2F_SDK_PATH` | SDK 경로 | `/usr/local` |
| `A2F_LIB_PATH` | 라이브러리 경로 | `/usr/local/lib` |
| `SKIP_TRT_BUILD` | TRT 빌드 건너뛰기 | - |
| `FORCE_TRT_REBUILD` | TRT 강제 재빌드 | - |

## 트러블슈팅

### Pod이 Pending 상태로 유지됨

```bash
kubectl describe pod a2f-grpc-server-0
```

**원인 1: GPU 노드 없음**
```
Warning  FailedScheduling  Insufficient nvidia.com/gpu
```
→ GPU 노드 그룹 생성 필요

**원인 2: nodeSelector 불일치**
```
Warning  FailedScheduling  node(s) didn't match Pod's node affinity/selector
```
→ 노드 레이블 확인: `kubectl get nodes --show-labels`

**원인 3: volume node affinity conflict**
```
Warning  FailedScheduling  node(s) had volume node affinity conflict
```
→ PVC가 다른 AZ에 바인딩됨. StatefulSet의 volumeClaimTemplates는 Pod별 독립 PVC를 생성하므로 이 문제가 발생하지 않아야 함. 기존 PVC가 남아있다면 정리 필요.

### TRT 빌드 실패

```bash
kubectl logs a2f-grpc-server-0 | grep -i error
```

**원인: ONNX 파일 없음**
```
[entrypoint] ONNX not found: /data/models/multi-diffusion/network.onnx
```
→ S3 모델 데이터 확인: `aws s3 ls s3://avadot/a2f-models/`

**원인: GPU 메모리 부족**
```
[TRT] [E] Out of memory
```
→ 더 큰 GPU 인스턴스 사용 (g4dn.xlarge → g4dn.2xlarge)

### gRPC 연결 실패

```bash
# 서비스 엔드포인트 확인
kubectl get endpoints a2f-grpc-server

# Pod 내부에서 테스트
kubectl run grpc-test --rm -it --image=fullstorydev/grpcurl \
  -- -plaintext a2f-grpc-server:50051 list
```

### Startup Probe 실패

TRT 빌드가 예상보다 오래 걸리는 경우 (현재 최대 10분):

```bash
# Startup probe 타임아웃 증가
kubectl patch statefulset a2f-grpc-server --type='json' -p='[
  {"op": "replace", "path": "/spec/template/spec/containers/0/startupProbe/failureThreshold", "value": 72}
]'
```

## 모델 디렉토리 구조

각 Pod의 PVC 내부 구조:

```
/data/models/
├── multi-diffusion/         # 영어 모델 (Claire, James, Mark)
│   ├── model.json
│   ├── network.onnx         # 원본 모델
│   ├── network.trt          # 빌드된 TRT 엔진 (자동 생성)
│   ├── .gpu_arch            # GPU 아키텍처 마커 (자동 생성)
│   └── ...
└── a23/                     # 한국어 모델
    ├── model.json
    ├── network.onnx
    ├── network.trt          # 자동 생성
    ├── .gpu_arch            # 자동 생성
    └── ...
```

## 참고

- [Docker 설정](../../a2f-grpc-server/docker/Dockerfile)
- [Entrypoint 스크립트](../../a2f-grpc-server/docker/entrypoint.sh)
- [A2F gRPC Server CLAUDE.md](../../a2f-grpc-server/CLAUDE.md)
