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
┌─────────────────────────────────────────────────────────────┐
│                     EKS Cluster                              │
│                                                              │
│  ┌────────────────────┐      ┌────────────────────────────┐ │
│  │  Backend Pod       │      │  GPU Node (avadot-node-gpu)│ │
│  │  (CPU Node)        │      │                            │ │
│  │                    │      │  ┌──────────────────────┐  │ │
│  │  avadot-backend    │─────▶│  │ a2f-grpc-server Pod  │  │ │
│  │  :4008             │ gRPC │  │ :50051               │  │ │
│  │                    │      │  │                      │  │ │
│  └────────────────────┘      │  │ ┌──────────────────┐ │  │ │
│                              │  │ │ PVC: model-data  │ │  │ │
│                              │  │ │ /data/models     │ │  │ │
│                              │  │ └──────────────────┘ │  │ │
│                              │  └──────────────────────┘  │ │
│                              └────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

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
- 이후 배포: **즉시** (캐시된 TRT 사용)

### PVC 요구사항

모델 데이터 PVC는 **쓰기 가능**해야 합니다:
- TRT 엔진 파일 저장 (`network.trt`)
- GPU 아키텍처 마커 파일 저장 (`.gpu_arch`)

## 배포 전 준비사항

### 1. GPU 노드 그룹 생성

```bash
# eksctl로 GPU 노드 그룹 추가
eksctl create nodegroup \
  --cluster avadot \
  --name avadot-node-gpu \
  --node-type g4dn.xlarge \
  --nodes 1 \
  --nodes-min 1 \
  --nodes-max 2 \
  --node-labels "alpha.eksctl.io/nodegroup-name=avadot-node-gpu" \
  --install-nvidia-plugin
```

### 2. 모델 데이터 업로드

PVC 생성 후 모델 데이터를 업로드해야 합니다:

```bash
# 임시 Pod으로 PVC에 데이터 복사
kubectl run data-loader --image=busybox --restart=Never \
  --overrides='{"spec":{"containers":[{"name":"data-loader","image":"busybox","command":["sleep","3600"],"volumeMounts":[{"name":"data","mountPath":"/data"}]}],"volumes":[{"name":"data","persistentVolumeClaim":{"claimName":"a2f-model-data"}}]}}'

# 로컬에서 모델 데이터 복사
kubectl cp ./models/multi-diffusion data-loader:/data/models/multi-diffusion
kubectl cp ./models/a23 data-loader:/data/models/a23

# 임시 Pod 삭제
kubectl delete pod data-loader
```

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

# 배포 상태 확인
kubectl get pods -l app=a2f-grpc-server -w
```

### 로그 확인

```bash
# TRT 빌드 로그 (첫 배포 시)
kubectl logs -l app=a2f-grpc-server -f

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
kubectl describe pod -l app=a2f-grpc-server
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

### TRT 빌드 실패

```bash
kubectl logs -l app=a2f-grpc-server | grep -i error
```

**원인: ONNX 파일 없음**
```
[entrypoint] ONNX not found: /data/models/multi-diffusion/network.onnx
```
→ 모델 데이터 업로드 확인

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

### Startup Probe 실패 (300초 후)

TRT 빌드가 예상보다 오래 걸리는 경우:

```bash
# Startup probe 타임아웃 증가
kubectl patch deployment a2f-grpc-server --type='json' -p='[
  {"op": "replace", "path": "/spec/template/spec/containers/0/startupProbe/failureThreshold", "value": 36}
]'
```

### PVC 권한 문제

```bash
kubectl logs -l app=a2f-grpc-server | grep -i permission
```

TRT 파일 쓰기 실패 시:
→ PVC가 ReadWriteOnce인지 확인
→ 볼륨이 `:ro`로 마운트되지 않았는지 확인

## 모델 디렉토리 구조

PVC 내부 예상 구조:

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
