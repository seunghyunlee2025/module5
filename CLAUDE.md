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
