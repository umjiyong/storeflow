전체 2개월 구조

1개월차: 핵심 신뢰성 확보

앞서 설계한 범위.

주문·결제 도메인
모듈러 모놀리스
결제 멱등성
동시 요청 제어
결제 UNKNOWN 상태
Kafka
Transactional Outbox
Consumer 멱등성
외부 결제 장애 실험
Prometheus·Grafana
1차 부하 테스트
ADR·장애 보고서

1개월차 종료 시점에는 작지만 깊이 있는 결제 시스템이 있어야 한다.

2개월차: 제품 범위·분산 구조·운영 수준 확장

두 번째 달은 기능을 마구 추가하는 달이 아니라 다음 네 방향으로 확장해야 함.

결제에서 매장 운영 제품으로 범위 확장
단일 애플리케이션에서 확장 가능한 구조로 진화
정상 처리에서 복합 장애와 복구로 심화
개발 저장소에서 지원 가능한 포트폴리오로 완성

-----
한 달 종료 시 필요한 최소 산출물

README.md
docs/
├── 01-problem-statement.md
├── 02-business-rules.md
├── 03-domain-model.md
├── 04-architecture.md
├── adr/
│   ├── 001-modular-monolith.md
│   ├── 002-idempotency.md
│   ├── 003-payment-unknown-state.md
│   └── 004-outbox-pattern.md
├── incidents/
│   ├── pg-timeout.md
│   ├── kafka-outage.md
│   └── application-crash.md
├── performance/
│   ├── baseline.md
│   └── optimization-result.md
└── retrospective.md

그리고 실행 명령 하나로 환경이 올라와야 함.

docker compose up
