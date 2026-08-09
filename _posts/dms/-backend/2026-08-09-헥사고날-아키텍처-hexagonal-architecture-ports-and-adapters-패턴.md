---
layout: post
title: "[Daily morning study] 헥사고날 아키텍처 (Hexagonal Architecture) — Ports and Adapters 패턴"
description: >
  #daily morning study
category: 
    - dms
    - dms-backend
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 헥사고날 아키텍처란

헥사고날 아키텍처(Hexagonal Architecture)는 Alistair Cockburn이 2005년에 제안한 소프트웨어 설계 패턴이다. **Ports and Adapters 패턴**이라고도 부른다.

핵심 목표는 하나다. **비즈니스 로직(도메인)을 외부 의존성으로부터 완전히 격리**하는 것이다. DB가 MySQL이든 PostgreSQL이든, HTTP API든 CLI든 도메인 코드는 바뀌지 않아야 한다.

육각형(Hexagon) 모양은 특별한 의미가 아니라 "여러 방향에서 들어오고 나가는 포트가 있다"는 걸 시각적으로 표현한 것이다.

---

## 전통적인 레이어드 아키텍처의 문제

일반적인 3-tier 아키텍처는 이렇게 생겼다.

```
Controller → Service → Repository → DB
```

문제는 방향성이 한쪽으로만 흐르고, **도메인 로직이 DB 구조에 종속**되기 쉽다는 점이다.

- 테스트할 때 DB가 없으면 서비스 레이어를 테스트하기 어렵다
- DB를 바꾸면 서비스 코드까지 수정해야 할 수 있다
- HTTP 요청이 아닌 CLI나 메시지 큐에서 같은 로직을 재사용하기 불편하다

헥사고날 아키텍처는 이 문제를 **인터페이스(Port)** 로 해결한다.

---

## 구성 요소

### 1. 도메인 (Domain / Application Core)

비즈니스 로직이 담긴 중심부다. 외부를 전혀 몰라야 한다. DB도, HTTP도, 프레임워크도 의존하지 않는다.

### 2. 포트 (Port)

도메인이 외부와 소통하는 **인터페이스**다. 두 종류가 있다.

| 포트 종류 | 방향 | 역할 | 예시 |
|-----------|------|------|------|
| Driving Port (Inbound) | 외부 → 도메인 | 외부가 도메인을 호출하는 계약 | `OrderService` 인터페이스 |
| Driven Port (Outbound) | 도메인 → 외부 | 도메인이 외부를 호출하는 계약 | `OrderRepository` 인터페이스 |

### 3. 어댑터 (Adapter)

포트를 실제로 구현하는 코드다. 외부 세계와 도메인 사이의 **변환기** 역할을 한다.

| 어댑터 종류 | 구현 대상 | 예시 |
|-------------|-----------|------|
| Driving Adapter | Driving Port | REST Controller, CLI Handler, Kafka Consumer |
| Driven Adapter | Driven Port | JPA Repository, Redis Client, HTTP Client |

---

## 구조 시각화

```
                ┌──────────────────────────────────┐
                │                                  │
  REST API ──── │──► OrderController               │
                │         │                        │
  CLI ───────── │──► CliHandler                    │
                │         │                        │
                │    ┌────▼────────────────────┐   │
                │    │    OrderService (Port)  │   │
                │    │                         │   │
                │    │   [비즈니스 로직]        │   │
                │    │                         │   │
                │    │  OrderRepository (Port) │   │
                │    └────┬────────────────────┘   │
                │         │                        │
                │    ┌────▼────────────────────┐   │
                │    │  JpaOrderRepository     │   │──► DB
                │    │  (Driven Adapter)       │   │
                │    └─────────────────────────┘   │
                │                                  │
                └──────────────────────────────────┘
```

---

## 코드 예시 (Java / Spring)

### Driving Port — 서비스 인터페이스

```java
// domain/port/in/PlaceOrderUseCase.java
public interface PlaceOrderUseCase {
    OrderId placeOrder(PlaceOrderCommand command);
}
```

### Driven Port — 저장소 인터페이스

```java
// domain/port/out/OrderRepository.java
public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(OrderId id);
}
```

### 도메인 서비스 (외부 의존 없음)

```java
// domain/service/OrderService.java
@Service
public class OrderService implements PlaceOrderUseCase {

    private final OrderRepository orderRepository; // Port에만 의존

    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    @Override
    public OrderId placeOrder(PlaceOrderCommand command) {
        Order order = Order.create(command.getProductId(), command.getQuantity());
        orderRepository.save(order);
        return order.getId();
    }
}
```

### Driving Adapter — REST 컨트롤러

```java
// adapter/in/web/OrderController.java
@RestController
@RequestMapping("/orders")
public class OrderController {

    private final PlaceOrderUseCase placeOrderUseCase; // Port만 알면 된다

    public OrderController(PlaceOrderUseCase placeOrderUseCase) {
        this.placeOrderUseCase = placeOrderUseCase;
    }

    @PostMapping
    public ResponseEntity<OrderId> placeOrder(@RequestBody PlaceOrderRequest request) {
        OrderId orderId = placeOrderUseCase.placeOrder(
            new PlaceOrderCommand(request.getProductId(), request.getQuantity())
        );
        return ResponseEntity.ok(orderId);
    }
}
```

### Driven Adapter — JPA 저장소 구현

```java
// adapter/out/persistence/JpaOrderRepository.java
@Repository
public class JpaOrderRepository implements OrderRepository { // Port를 구현

    private final OrderJpaRepository jpaRepository; // Spring Data JPA

    public JpaOrderRepository(OrderJpaRepository jpaRepository) {
        this.jpaRepository = jpaRepository;
    }

    @Override
    public void save(Order order) {
        OrderEntity entity = OrderMapper.toEntity(order);
        jpaRepository.save(entity);
    }

    @Override
    public Optional<Order> findById(OrderId id) {
        return jpaRepository.findById(id.getValue())
            .map(OrderMapper::toDomain);
    }
}
```

---

## 테스트가 쉬워지는 이유

도메인이 인터페이스(Port)에만 의존하기 때문에, 테스트에서 **실제 DB 없이 Mock으로 교체**할 수 있다.

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private OrderRepository orderRepository; // DB 없이 Mock 사용

    @InjectMocks
    private OrderService orderService;

    @Test
    void 주문_생성_성공() {
        PlaceOrderCommand command = new PlaceOrderCommand("PRODUCT-1", 2);

        OrderId result = orderService.placeOrder(command);

        assertNotNull(result);
        verify(orderRepository).save(any(Order.class));
    }
}
```

---

## 패키지 구조 예시

```
src/main/java/com/example/
├── domain/
│   ├── model/              # Order, OrderId, Product 등 도메인 객체
│   ├── service/            # OrderService (비즈니스 로직)
│   └── port/
│       ├── in/             # Driving Port (UseCase 인터페이스)
│       └── out/            # Driven Port (Repository 인터페이스)
│
└── adapter/
    ├── in/
    │   ├── web/            # REST Controller
    │   └── messaging/      # Kafka Consumer
    └── out/
        ├── persistence/    # JPA Repository 구현체
        └── notification/   # 이메일/SMS 전송 구현체
```

---

## 레이어드 아키텍처 vs 헥사고날 아키텍처 비교

| 항목 | 레이어드 아키텍처 | 헥사고날 아키텍처 |
|------|-------------------|-------------------|
| 의존성 방향 | 위 → 아래 (단방향) | 외부 → 도메인 (안쪽 방향) |
| 도메인 격리 | 약함 (DB 구조에 영향받음) | 강함 (외부 의존 없음) |
| 테스트 용이성 | DB/프레임워크 필요한 경우 多 | Unit Test 독립적으로 가능 |
| 입력 채널 추가 | 구조 변경 필요할 수 있음 | Adapter만 추가하면 됨 |
| 학습 곡선 | 낮음 | 높음 (추상화 레이어 多) |
| 소규모 프로젝트 | 적합 | 오버엔지니어링일 수 있음 |

---

## DDD와의 관계

헥사고날 아키텍처는 DDD와 자주 함께 쓰인다. DDD가 **무엇을 모델링할지** 알려준다면, 헥사고날 아키텍처는 **어떻게 구조를 잡을지** 알려준다.

- DDD의 Aggregate, Entity, Value Object → 도메인 모델로 구현
- DDD의 Repository → Driven Port로 정의, 어댑터에서 구현
- DDD의 Application Service → Driving Port(UseCase)로 노출

---

## 언제 적용하면 좋은가

- 비즈니스 로직이 복잡하고 장기적으로 유지보수해야 하는 시스템
- 여러 입력 채널(HTTP, CLI, 메시지 큐)을 동시에 지원해야 하는 경우
- DB나 외부 서비스를 나중에 교체할 가능성이 있는 경우
- 도메인 로직을 독립적으로 단위 테스트해야 하는 경우

반대로 단순한 CRUD 앱이나 프로토타입 단계에서는 오히려 복잡도만 높아질 수 있다.

---

## 핵심 정리

- 헥사고날 아키텍처 = **도메인을 외부로부터 격리**하는 Ports and Adapters 패턴
- **Port** = 도메인이 외부와 소통하는 인터페이스 (Driving / Driven 두 종류)
- **Adapter** = 포트를 실제로 구현하는 코드 (REST 컨트롤러, JPA, Kafka 등)
- 도메인은 프레임워크, DB, HTTP를 전혀 모른다 → **테스트 용이, 기술 교체 쉬움**
- DDD와 궁합이 좋으며, 복잡한 비즈니스 도메인에 적합하다
