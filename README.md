# StoreFlow

오프라인 매장의 주문과 결제 과정에서 발생할 수 있는  
**중복 요청, 네트워크 장애, 데이터 정합성 문제를 안정적으로 처리하기 위한 서버 프로젝트**입니다.

단순히 결제 API를 구현하는 것이 아니라, 오프라인 환경에서 실제로 발생할 수 있는 장애 상황과 운영 문제를 직접 정의하고, 이를 설계·구현·테스트·관측하는 과정을 통해 확장 가능하고 안정적인 서버 구조를 학습하는 것을 목표로 합니다.

---

## Why StoreFlow?

오프라인 결제 환경에서는 온라인 서비스와 다른 문제가 발생할 수 있습니다.

POS 또는 결제 단말기는 항상 안정적인 네트워크 환경에 연결되어 있다고 가정할 수 없으며, 사용자는 결제 결과를 받지 못했을 때 동일한 요청을 다시 시도할 수 있습니다.

예를 들어 다음과 같은 상황을 생각할 수 있습니다.

1. POS에서 결제를 요청합니다.
2. 서버가 외부 결제 시스템에 승인을 요청합니다.
3. 실제 결제 승인은 성공합니다.
4. 네트워크 문제로 POS가 성공 응답을 받지 못합니다.
5. POS가 동일한 결제를 다시 요청합니다.

이 상황을 단순하게 처리하면 동일 주문에 대해 중복 결제가 발생할 수 있습니다.

StoreFlow에서는 이러한 문제를 단순 예외 처리로 해결하기보다, 결제 도메인의 상태와 시스템 간 데이터 흐름을 명확하게 설계하고 장애 상황에서도 일관된 결과를 만들 수 있는 구조를 고민합니다.

---

## Project Goals

이 프로젝트에서는 다음 문제를 중심으로 시스템을 발전시킵니다.

### 1. 중복 결제 방지

네트워크 지연이나 응답 유실로 동일한 결제 요청이 반복되더라도 실제 결제 승인은 한 번만 수행되어야 합니다.

이를 위해 다음 주제를 다룹니다.

- Idempotency Key
- DB Unique Constraint
- 동시 요청 제어
- Redis를 활용한 임시 상태 관리
- 결제 결과 재조회

---

### 2. 불확실한 결제 상태 처리

외부 결제 시스템의 승인 요청이 성공했는지 실패했는지 즉시 판단할 수 없는 상황이 존재할 수 있습니다.

따라서 결제를 단순히 성공과 실패 두 상태로만 표현하지 않고 다음과 같이 불확실한 상태를 명시적으로 표현합니다.

```text
READY
  ↓
PROCESSING
  ↓
APPROVED

PROCESSING
  ↓
FAILED

PROCESSING
  ↓
UNKNOWN
```

`UNKNOWN` 상태의 결제는 이후 조회 또는 복구 프로세스를 통해 최종 상태를 결정합니다.

---

### 3. 데이터 저장과 이벤트 발행의 정합성

결제가 성공하면 이후 여러 시스템에서 해당 결과를 사용할 수 있습니다.

예를 들어 다음과 같은 후속 처리가 발생할 수 있습니다.

```text
Payment Approved
       │
       ├── Order
       ├── Settlement
       ├── Inventory
       └── Notification
```

하지만 다음과 같은 구현에서는 문제가 발생할 수 있습니다.

```text
1. 결제 정보 DB 저장 성공
2. Kafka 이벤트 발행 실패
```

이 경우 DB에는 결제가 완료되었지만 다른 시스템은 결제 완료 사실을 알지 못하게 됩니다.

StoreFlow에서는 이러한 문제를 직접 재현한 뒤 `Transactional Outbox Pattern`을 적용하여 DB 상태와 이벤트 전달 사이의 정합성을 개선합니다.

---

### 4. 장애를 고려한 시스템 설계

정상적인 요청만 처리하는 시스템이 아니라 다음과 같은 장애 상황을 직접 만들어 테스트합니다.

- 외부 결제 시스템 응답 지연
- 결제 승인 이후 응답 유실
- 동일 결제 동시 요청
- Kafka 장애
- Consumer 중복 처리
- Application Process 종료
- DB Connection Pool 고갈
- Redis 장애

장애 발생 시 다음 흐름으로 문제를 기록합니다.

```text
현상
→ 사용자 영향
→ 시스템 지표
→ 원인 분석
→ 복구
→ 구조 개선
```

---

## Development Strategy

StoreFlow는 처음부터 복잡한 분산 시스템으로 시작하지 않습니다.

초기에는 하나의 애플리케이션 내부에서 도메인 경계를 명확하게 나누는 **Modular Monolith** 구조로 시작합니다.

```text
storeflow
├── order
│   ├── domain
│   ├── application
│   └── infrastructure
│
├── payment
│   ├── domain
│   ├── application
│   └── infrastructure
│
├── inventory
│   ├── domain
│   ├── application
│   └── infrastructure
│
├── settlement
│   ├── domain
│   ├── application
│   └── infrastructure
│
└── support
```

시스템 규모에 비해 불필요한 복잡도를 먼저 도입하지 않고, 실제 문제가 발생하거나 구조적 필요성이 확인되었을 때 기술과 아키텍처를 단계적으로 추가합니다.

---

## Architecture Evolution

프로젝트는 다음 순서로 발전할 예정입니다.

### Phase 1 — Core Payment System

```text
POS
 ↓
StoreFlow
 ↓
Payment Simulator
```

주요 목표:

- 주문 생성
- 결제 요청
- 결제 상태 관리
- Idempotency
- 동시 요청 제어
- 외부 결제 장애 처리

---

### Phase 2 — Event Driven Architecture

```text
              ┌─────────────┐
              │    Order    │
              └─────────────┘
                     ↑
                     │
POS → Payment → Kafka → Settlement
                     │
                     ↓
                 Inventory
```

주요 목표:

- Kafka Event
- Transactional Outbox
- Consumer Idempotency
- Retry
- DLQ
- Event Contract

---

### Phase 3 — Store Operation

결제를 중심으로 실제 매장 운영 흐름까지 확장합니다.

```text
Store
 ↓
Product
 ↓
Order
 ↓
Inventory
 ↓
Payment
 ↓
Refund
 ↓
Settlement
```

주요 목표:

- 매장별 상품
- 주문 시점 가격 Snapshot
- 재고 동시성
- 결제 취소
- 부분 취소
- 주문 만료
- 매출 원장
- 일별 정산

---

### Phase 4 — Scale & Reliability

시스템의 기능이 완성된 이후 성능과 운영 안정성을 검증합니다.

주요 목표:

- Load Test
- Bottleneck Analysis
- Prometheus Metrics
- Grafana Dashboard
- Kafka Consumer Lag
- Outbox Backlog
- Failure Injection
- Kubernetes Deployment
- Rolling Update

---

## Tech Stack

프로젝트가 진행됨에 따라 필요한 기술을 단계적으로 도입합니다.

### Application

- Java
- Spring Boot
- Spring Data JPA
- Hibernate

### Data

- MySQL
- Redis

### Messaging

- Apache Kafka

### Testing

- JUnit
- Testcontainers
- k6 / Gatling

### Observability

- Spring Boot Actuator
- Micrometer
- Prometheus
- Grafana

### Infrastructure

- Docker
- Docker Compose
- Kubernetes
- ArgoCD

> 모든 기술은 단순히 사용 경험을 만들기 위해 도입하지 않습니다.  
> 해결하려는 문제가 존재하는 경우에만 도입하고, 선택 이유와 트레이드오프를 ADR로 기록합니다.

---

## Key Engineering Topics

이 프로젝트에서 특히 깊게 다루고자 하는 주제입니다.

### Payment

- Payment Idempotency
- Payment State Machine
- Payment UNKNOWN State
- Duplicate Payment Prevention
- Refund Idempotency

### Database

- Transaction Boundary
- Optimistic Lock
- Pessimistic Lock
- Conditional Update
- Unique Constraint
- Index Design

### Distributed System

- Event Driven Architecture
- Transactional Outbox
- At-least-once Delivery
- Idempotent Consumer
- Retry & DLQ
- Event Ordering
- Event Versioning

### Reliability

- Timeout
- Retry
- Failure Isolation
- Graceful Recovery
- Back Pressure
- Connection Pool Exhaustion

### Observability

- Business Metrics
- Technical Metrics
- Structured Logging
- Latency
- Error Rate
- Consumer Lag

---

## Engineering Principles

### Solve Problems Before Adding Technology

새로운 기술은 프로젝트의 목적이 아닙니다.

먼저 문제를 재현하고 현재 구조에서 해결할 수 있는지 검토한 뒤, 필요성이 확인된 경우에만 새로운 기술을 도입합니다.

---

### Failure Is Part of the System

외부 시스템, 네트워크, 메시지 브로커, 데이터베이스는 언제든 실패할 수 있다고 가정합니다.

정상 경로뿐 아니라 실패 경로도 시스템의 주요 동작으로 간주합니다.

---

### Make Decisions Explicit

주요 설계 판단은 ADR(Architecture Decision Record)로 기록합니다.

```text
Problem
Context
Options
Decision
Trade-off
Result
```

이를 통해 코드뿐 아니라 왜 현재 구조가 만들어졌는지를 추적할 수 있도록 합니다.

---

### Verify With Experiments

설계가 올바르다고 가정하지 않습니다.

동시성 테스트, 장애 주입, 부하 테스트, 메트릭 분석을 통해 실제로 의도한 방식으로 동작하는지 확인합니다.

---

## Documentation

```text
docs/

├── product/
│   ├── problem-statement.md
│   ├── user-scenario.md
│   └── business-rules.md
│
├── architecture/
│   ├── system-context.md
│   ├── domain-model.md
│   ├── event-flow.md
│   └── architecture-evolution.md
│
├── adr/
│   ├── 001-modular-monolith.md
│   ├── 002-payment-idempotency.md
│   ├── 003-payment-unknown-state.md
│   └── 004-transactional-outbox.md
│
├── incidents/
│
├── performance/
│
└── retrospective/
```

---

## Roadmap

### Month 1 — Payment Reliability

- [ ] Order Domain
- [ ] Payment Domain
- [ ] Payment Simulator
- [ ] Payment Idempotency
- [ ] Concurrent Payment Test
- [ ] Payment UNKNOWN State
- [ ] Kafka
- [ ] Transactional Outbox
- [ ] Idempotent Consumer
- [ ] Prometheus
- [ ] Grafana
- [ ] Failure Injection
- [ ] Load Test

### Month 2 — Product & Scale

- [ ] Refund
- [ ] Partial Refund
- [ ] Store
- [ ] Product
- [ ] Inventory
- [ ] Inventory Concurrency
- [ ] Order Expiration
- [ ] Sales Ledger
- [ ] Settlement
- [ ] Service Separation
- [ ] Event Versioning
- [ ] Kubernetes
- [ ] ArgoCD
- [ ] Advanced Load Test
- [ ] Failure Recovery Test
- [ ] Performance Optimization

---

## Current Status

🚧 **Project Started**

현재 프로젝트는 초기 설계 단계입니다.

첫 번째 목표는 복잡한 기능을 추가하는 것이 아니라 다음 질문에 답하는 것입니다.

> 동일한 결제 요청이 여러 번 전달되더라도 하나의 결제만 안전하게 처리할 수 있는가?

프로젝트가 진행되면서 각 설계 판단과 실패 과정, 성능 테스트 결과를 지속적으로 기록합니다.

---

## Disclaimer

StoreFlow는 실제 금융 서비스를 제공하기 위한 시스템이 아닙니다.

실제 카드사, VAN, PG와 연결하지 않고 외부 결제 시스템을 모사한 `Payment Simulator`를 사용합니다.

따라서 실제 결제 서비스의 보안, 규제, 인증, 금융 인프라 전체를 구현하는 것이 목적이 아닙니다.

이 프로젝트의 목적은 오프라인 결제 및 매장 운영 시스템에서 발생할 수 있는 서버 설계와 운영 문제를 학습하고 검증하는 것입니다.
