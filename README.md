# 시스템 리소스 모니터링

서버의 CPU, 메모리, 디스크, 네트워크를 실시간으로 모니터링하고 시각화하는 시스템입니다.

## 빠른 시작

### 1. 설치
```bash
npm install
```

### 2. 환경 설정
```bash
cp .env.example .env
# .env 파일을 편집하여 설정값 입력
```

### 3. 실행
```bash
# 전체 시스템 실행 (Agent + Server)
npm run dev

# 또는 개별 실행
npm run agent   # Collector Agent만 실행
npm run server  # API Server만 실행
```

## 주요 기능

- **실시간 메트릭 수집**: CPU, 메모리, 디스크, 네트워크 사용량
- **시계열 데이터 저장**: InfluxDB를 사용한 효율적인 데이터 저장
- **대시보드 시각화**: Grafana를 통한 직관적인 모니터링
- **임계값 기반 알림**: Slack, 이메일로 실시간 알림
- **멀티 서버 지원**: 여러 서버의 메트릭을 중앙에서 관리

## 아키텍처

```
[Collector Agent] → [API Server] → [InfluxDB] → [Grafana]
                         ↓
                    [Alert Engine] → [Slack/Email]
```

## 문서

- [PRD (제품 요구사항 정의서)](./docs/system-resource-monitoring-prd.md)
- [CLAUDE.md (개발 가이드)](./CLAUDE.md)

## 테스트

```bash
npm test              # 전체 테스트
npm run test:coverage # 커버리지 확인
```

## 라이선스

ISC
