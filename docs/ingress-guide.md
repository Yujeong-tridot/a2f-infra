# EKS Ingress 설정 가이드

## 개요
EKS 클러스터에 ALB Ingress를 설정하기 위한 논의사항 및 체크리스트입니다.

## 현재 환경
| 항목 | 상태 |
|------|------|
| 클러스터 | EKS |
| Load Balancer | AWS ALB (Ingress Controller 설치됨) |
| 서비스 | backend(4008), tarot(4007/WS), blind-date(4010/WS) |
| 기존 도메인 | avadot.com (EC2 staging, Namecheap SSL) |

---

## 1. 라우팅 방식 결정

### 옵션 A: 경로 기반 (Path-based)
```
ALB-IP/api/*        → backend:4008
ALB-IP/tarot/*      → tarot:4007
ALB-IP/blind-date/* → blind-date:4010
```
- ✅ 단일 ALB로 비용 절감
- ✅ IP만으로 테스트 가능
- ⚠️ 서비스가 해당 prefix를 인식해야 함 (또는 rewrite 설정 필요)

### 옵션 B: 서브도메인 기반
```
api.avadot.com    → backend:4008
tarot.avadot.com  → tarot:4007
blind.avadot.com  → blind-date:4010
```
- ✅ 깔끔한 서비스 분리
- ⚠️ 도메인 필요 (IP만으로는 불가)

### 📋 확인 필요
- [ ] 각 서비스의 API 경로 구조 확인 (prefix 포함 여부)
- [ ] 프론트엔드 API 호출 방식 확인
- [ ] rewrite 필요 여부 결정

---

## 2. Health Check 설정

### 현재 서비스별 설정
| 서비스 | Health Check 경로 | 프로토콜 |
|--------|------------------|----------|
| backend | `/` | HTTPS |
| tarot | `/health` | HTTPS |
| blind-date | `/` | HTTPS |

### 문제점
ALB Ingress는 기본적으로 **전역 health check 경로**를 사용합니다.
서비스마다 다른 경로 사용 시 추가 설정이 필요합니다.

### 📋 확인 필요
- [ ] 모든 서비스가 `/health` 또는 `/`로 통일 가능한지
- [ ] tarot 서비스 health check 경로 변경 가능 여부

---

## 3. WebSocket 지원 설정

tarot, blind-date 서비스가 WebSocket을 사용합니다.

### 필요한 ALB 설정
```yaml
# Idle timeout (WebSocket 연결 유지)
alb.ingress.kubernetes.io/load-balancer-attributes: idle_timeout.timeout_seconds=3600

# Sticky session (같은 Pod으로 연결 유지)
alb.ingress.kubernetes.io/target-group-attributes: stickiness.enabled=true,stickiness.lb_cookie.duration_seconds=3600

# Pod IP 직접 연결
alb.ingress.kubernetes.io/target-type: ip
```

### 📋 확인 필요
- [ ] WebSocket 연결 유지 시간 요구사항 (현재 설정: 1시간)
- [ ] 클라이언트 재연결 로직 존재 여부

---

## 4. SSL/HTTPS 설정

### 현재 상황
- 컨테이너들이 **자체 HTTPS** 서비스 중
- EC2 staging: Namecheap SSL 사용

### 옵션들

| 옵션 | 구조 | 설명 |
|------|------|------|
| A. ALB SSL 종료 | Client→HTTPS→ALB→HTTP→Pod | ACM 인증서, 컨테이너 HTTP 전환 필요 |
| B. End-to-End | Client→HTTPS→ALB→HTTPS→Pod | 현재 구조 유지, `backend-protocol: HTTPS` |
| C. HTTP only | Client→HTTP→ALB→HTTP/HTTPS→Pod | 테스트용, 가장 간단 |

### 📋 확인 필요
- [ ] 컨테이너가 HTTP로 서비스 가능한지 (설정 변경 필요?)
- [ ] ACM 인증서 발급할 도메인 결정
- [ ] IP 테스트 기간 중 HTTP만 사용할지

---

## 5. 도메인 전환 계획

### 현재
```
avadot.com → EC2 staging (Namecheap SSL)
```

### 전환 옵션
1. **새 서브도메인**: `eks.avadot.com` 또는 `api-v2.avadot.com`
2. **완전 전환**: avadot.com을 EKS로 이전
3. **별도 도메인**: 새 도메인 사용

### 📋 확인 필요
- [ ] EC2 staging과 EKS 공존 기간
- [ ] 도메인 전환 타이밍
- [ ] DNS 관리 방식 (Route53 vs Namecheap)

---

## 6. 배포 순서

### 권장 순서
1. Namespace, Secrets 생성
2. Deployment + Service 배포
3. Pod 정상 동작 확인 (`kubectl get pods`)
4. **Ingress 적용**
5. ALB 생성 확인 (AWS 콘솔)
6. Target Group healthy 확인
7. ALB DNS/IP로 접근 테스트

### 검증 명령어
```bash
# Ingress 상태 확인
kubectl get ingress -n avadot-prod

# ALB 주소 확인
kubectl get ingress -n avadot-prod -o jsonpath='{.items[0].status.loadBalancer.ingress[0].hostname}'

# 각 경로 테스트
curl http://<ALB-DNS>/api/health
curl http://<ALB-DNS>/tarot/health
curl http://<ALB-DNS>/blind-date/health
```

---

## 7. 결정 요약표

| 항목 | 옵션 | 권장 (테스트 단계) |
|------|------|-------------------|
| 라우팅 | 경로 vs 서브도메인 | **경로 기반** |
| SSL | HTTP / HTTPS | **HTTP only** |
| Health check | 개별 vs 통일 | **`/` 통일** |
| 도메인 | 즉시 vs 나중 | **나중에 연결** |

---

## 다음 단계
1. [ ] 위 체크리스트 항목 팀 논의
2. [ ] 서비스 API 경로 구조 확인
3. [ ] Health check 경로 통일 여부 결정
4. [ ] IP 테스트용 HTTP Ingress 적용
5. [ ] 도메인/HTTPS는 안정화 후 추가
