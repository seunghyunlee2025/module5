# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 프로젝트 개요

시스템 리소스 모니터링 시스템 - 서버의 CPU, 메모리, 디스크, 네트워크를 실시간으로 수집하고 시계열 데이터베이스에 저장하여 Grafana로 시각화하는 모니터링 플랫폼입니다.

## 아키텍처

### 주요 컴포넌트
```
Collector Agent (Node.js)
    ↓ HTTP/REST API
Metric API Server (Express)
    ↓ InfluxDB Client
Time-Series Database (InfluxDB)
    ↓ Query
Visualization (Grafana)
```

### 데이터 플로우
1. **Collector Agent**: `systeminformation` 라이브러리를 사용하여 시스템 메트릭 수집 (10초 주기)
2. **API Server**: REST API로 메트릭 수신 및 검증
3. **InfluxDB**: 시계열 데이터 저장 (7일 원본, 30일 집계)
4. **Alert Engine**: 임계값 기반 알림 트리거 (CPU > 80%, Memory > 85% 등)
5. **Grafana**: 대시보드 시각화 및 알림 전송

## 기술 스택

### 경량형 조합 (소규모 서버 모니터링, 빠른 구축)

#### Core Runtime
- **Node.js**: 18+ LTS
  - 비동기 I/O 효율성, 크로스 플랫폼 지원
  - 단일 언어로 Agent와 Server 구현

#### 메트릭 수집
- **systeminformation** ^5.21.20
  - 크로스 플랫폼 시스템 정보 수집 (Windows, Linux, macOS)
  - CPU, 메모리, 디스크, 네트워크 통합 API

#### API 서버
- **Express** ^4.18.2
  - 경량 웹 프레임워크, 미들웨어 생태계
  - REST API 엔드포인트 구현
- **Joi** ^17.11.0
  - 메트릭 데이터 스키마 검증
  - 런타임 타입 체크
- **dotenv** ^16.3.1
  - 환경 변수 관리

#### 시계열 데이터베이스
- **InfluxDB** 2.x
  - 시계열 데이터 최적화 스토리지
  - Flux 쿼리 언어, 자동 데이터 보존 정책
- **@influxdata/influxdb-client** ^1.33.2
  - 공식 Node.js 클라이언트
  - 배치 쓰기 최적화

#### HTTP 클라이언트
- **axios** ^1.6.2
  - Promise 기반 HTTP 요청
  - Collector → API Server 메트릭 전송

#### 시각화
- **Grafana** OSS (최신 버전)
  - 대시보드 구성, 시계열 그래프
  - InfluxDB 네이티브 연동

#### 알림
- **Slack Webhook**
  - 실시간 알림 전송 (CPU/메모리 임계값 초과)
- **Nodemailer** (권장)
  - SMTP 기반 이메일 알림

#### 개발 도구
- **Jest** ^29.7.0
  - 단위/통합 테스트 프레임워크
  - 코드 커버리지 측정
- **Supertest** ^6.3.3
  - HTTP API 테스트
- **ESLint** ^8.54.0
  - 코드 품질 검사, 스타일 가이드
- **Prettier** ^3.1.0
  - 코드 포매터
- **concurrently** ^8.2.2
  - Agent와 Server 동시 실행 (개발 환경)

#### 배포
- **PM2** (권장)
  - Node.js 프로세스 관리자
  - 자동 재시작, 로그 관리, 클러스터 모드
- **systemd**
  - Linux 서비스 등록 (프로덕션)
  - 부팅 시 자동 시작

### 선택 이유
- **설치 간단**: npm install로 5분 내 설정 완료
- **메모리 효율적**: Agent < 100MB, Server < 200MB
- **학습 곡선 낮음**: JavaScript 단일 언어, 잘 알려진 도구들
- **무료 오픈소스**: 라이선스 비용 없음
- **크로스 플랫폼**: Windows, Linux, macOS 모두 지원

### 제약사항
- **서버 규모**: 1~50대 권장 (100대 초과 시 성능 저하)
- **데이터 보존**: 단일 노드 InfluxDB (클러스터링 미지원)
- **고가용성**: 단일 장애점 존재 (HA 구성 필요 시 엔터프라이즈 조합 검토)

## 개발 환경 설정

### 설치
```bash
npm install
```

### 환경 변수
`.env` 파일 생성 (`.env.example` 참고):
```bash
cp .env.example .env
```

주요 설정:
- `INFLUXDB_URL`, `INFLUXDB_TOKEN`, `INFLUXDB_ORG`, `INFLUXDB_BUCKET`: InfluxDB 연결 정보
- `API_PORT`, `API_KEY`: API 서버 설정
- `COLLECTION_INTERVAL`, `SERVER_ID`: Collector Agent 설정
- `SLACK_WEBHOOK_URL`: Slack 알림 설정
- `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASS`: 이메일 알림 설정
- `LOG_LEVEL`: 로그 레벨 (debug, info, warn, error)

### 로컬 개발 실행
```bash
# Collector Agent 실행
npm run agent

# API Server 실행
npm run server

# 전체 시스템 실행 (Agent + Server)
npm run dev
```

### 테스트
```bash
# 전체 테스트 실행
npm test

# Watch 모드로 테스트
npm run test:watch

# 테스트 커버리지 확인
npm run test:coverage

# 특정 테스트 파일 실행
npm test -- tests/unit/collectors/cpu.test.js
```

테스트는 `/tests/unit` (단위 테스트)와 `/tests/integration` (통합 테스트) 디렉토리로 구분됨.

### 린트 및 포맷
```bash
# ESLint 검사
npm run lint

# 자동 수정
npm run lint:fix

# Prettier 포맷팅
npm run format
```

## 코드 구조

### `/src/collectors/`
시스템 메트릭 수집 모듈. 각 collector는 독립적으로 동작하며 다음 인터페이스를 구현:
```javascript
{
  collect: async () => MetricData,
  validate: (data) => boolean
}
```

- `cpu.js`: CPU 사용률, 로드 애버리지, 코어별 통계
- `memory.js`: 메모리 사용량, 스왑, 버퍼/캐시
- `disk.js`: 디스크 사용률, I/O 처리량, IOPS
- `network.js`: 네트워크 트래픽, 패킷 통계, 연결 수

### `/src/api/`
REST API 엔드포인트 및 미들웨어:
- `routes/metrics.js`: POST `/api/v1/metrics` (메트릭 수신), GET `/api/v1/metrics` (조회)
- `middleware/auth.js`: API Key 또는 JWT 기반 인증
- `middleware/validation.js`: 메트릭 데이터 스키마 검증

### `/src/storage/`
InfluxDB 연동 레이어:
- `influxdb.js`: InfluxDB 클라이언트 래퍼, 배치 쓰기 최적화
- `queries.js`: 자주 사용하는 InfluxDB 쿼리 템플릿

### `/src/alerts/`
알림 시스템:
- `engine.js`: 메트릭 임계값 모니터링 및 알림 트리거
- `channels/`: Slack, Email, Discord 알림 채널 구현

### `/src/config/`
설정 관리:
- `default.js`: 기본 설정값 (수집 주기, 임계값 등)
- `thresholds.js`: 알림 임계값 정의

## 메트릭 데이터 구조

```javascript
{
  server_id: "web-server-01",
  timestamp: "2026-02-02T14:30:00Z",
  metrics: {
    cpu: { usage_percent, cores[], load_average[] },
    memory: { total_gb, used_gb, used_percent, available_gb, swap_used_percent },
    disk: { [mount]: { total_gb, used_gb, used_percent, read_mbps, write_mbps } },
    network: { [interface]: { rx_mbps, tx_mbps, rx_packets, tx_packets } }
  }
}
```

## 중요 알림 임계값

- CPU 사용률 > 80% (5분 지속) → 경고
- CPU 사용률 > 95% (1분 지속) → 긴급
- 메모리 사용률 > 85% → 경고
- 메모리 사용률 > 95% → 긴급
- 디스크 사용률 > 80% → 경고
- 네트워크 패킷 손실률 > 1% → 경고

## Grafana 대시보드

대시보드 JSON은 `/grafana/dashboards/` 디렉토리에 저장:
- `overview.json`: 전체 서버 개요
- `server-detail.json`: 서버별 상세 메트릭

## 성능 고려사항

- Collector Agent는 CPU < 5%, 메모리 < 100MB 유지
- InfluxDB 배치 쓰기: 최대 100개 메트릭 또는 5초마다
- API Rate Limiting: 서버당 초당 10 요청
- 메트릭 수집 실패 시 로컬 버퍼에 저장 후 재전송 (최대 1000개)

## 보안

- API 엔드포인트는 API Key 헤더 (`X-API-Key`) 필수
- 프로덕션에서는 HTTPS/TLS 통신 필수
- 프로세스 명령줄 인자에서 민감 정보 자동 필터링
- 서버 ID는 등록된 서버만 허용 (화이트리스트)

## 트러블슈팅

### InfluxDB 연결 실패
- `.env`의 `INFLUXDB_URL`, `INFLUXDB_TOKEN` 확인
- InfluxDB 서버 상태 확인: `curl http://localhost:8086/health`

### 메트릭 수집 안 됨
- `systeminformation` 패키지가 현재 OS 지원하는지 확인
- 로그 파일 확인: `logs/collector.log`

### 알림이 전송되지 않음
- Slack Webhook URL 또는 SMTP 설정 확인
- Alert Engine 로그 확인: `logs/alerts.log`
