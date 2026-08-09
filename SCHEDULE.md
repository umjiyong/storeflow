# 전체 2개월 구조

## 1개월 차: 핵심 신뢰성 확보

앞서 설계한 범위를 구현한다.

- 주문·결제 도메인
- 모듈러 모놀리스
- 결제 멱등성
- 동시 요청 제어
- 결제 `UNKNOWN` 상태
- Kafka
- Transactional Outbox
- Consumer 멱등성
- 외부 결제 장애 실험
- Prometheus·Grafana
- 1차 부하 테스트
- ADR·장애 보고서

> **1개월 차 종료 시점에는 작지만 깊이 있는 결제 시스템이 있어야 한다.**

---

## 2개월 차: 제품 범위·분산 구조·운영 수준 확장

두 번째 달은 기능을 마구 추가하는 달이 아니라 다음 네 방향으로 확장해야 한다.

1. 결제에서 **매장 운영 제품**으로 범위 확장
2. 단일 애플리케이션에서 **확장 가능한 구조**로 진화
3. 정상 처리에서 **복합 장애와 복구**로 심화
4. 개발 저장소에서 **지원 가능한 포트폴리오**로 완성

---

# 1개월 차 종료 시 필요한 최소 산출물

## 문서 구조

~~~text
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
~~~

## 실행 환경

다음 명령 하나로 전체 환경이 실행되어야 한다.

~~~bash
docker compose up
~~~

---

# 1개월 차 완료 기준

- [ ] 주문·결제 도메인이 명확하다.
- [ ] 결제 멱등성을 동시성 환경에서 검증했다.
- [ ] 응답 유실과 불확실한 결제 상태를 처리한다.
- [ ] Transactional Outbox Pattern으로 이벤트 유실 문제를 해결했다.
- [ ] Consumer의 중복 처리를 방지한다.
- [ ] 장애를 의도적으로 재현하고 복구했다.
- [ ] Prometheus와 Grafana에서 핵심 지표를 확인할 수 있다.
- [ ] 한 차례 이상의 부하 테스트와 성능 개선을 완료했다.
- [ ] 설계 판단과 실패 과정이 문서로 남아 있다.
- [ ] 전체 코드를 직접 설명할 수 있다.
