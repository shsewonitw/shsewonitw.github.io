---
layout: post
title: "[Daily morning study] 도메인 주도 설계 (Domain-Driven Design) 핵심 개념"
description: >
  #daily morning study
category: 
    - dms
    - dms-backend
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 도메인 주도 설계 (DDD)란

DDD는 복잡한 소프트웨어를 개발할 때 비즈니스 도메인을 중심에 놓고 설계하는 방법론이다. Eric Evans가 2003년 저서 "Domain-Driven Design"에서 제시했으며, 기술보다 도메인 전문가와의 협업과 도메인 지식을 코드에 녹여내는 데 초점을 맞춘다.

핵심 아이디어는 간단하다. 개발자와 도메인 전문가가 **같은 언어**로 소통하고, 그 언어가 코드에도 그대로 반영되어야 한다는 것이다.

---

## 유비쿼터스 언어 (Ubiquitous Language)

팀 전체(개발자 + 도메인 전문가)가 공유하는 공통 언어다.

- 코드의 클래스명, 메서드명, 변수명이 도메인 전문가가 사용하는 용어와 일치해야 한다
- 용어가 일치하지 않으면 번역 과정에서 오해가 생기고 버그로 이어진다

```java
// 나쁜 예 - 개발자식 명명
public class OrderProcessor {
    public void processData(int userId, List<Item> items) { ... }
}

// 좋은 예 - 유비쿼터스 언어 반영
public class Order {
    public void place(Customer customer, List<OrderLine> orderLines) { ... }
}
```

---

## 전략적 설계 (Strategic Design)

### 바운디드 컨텍스트 (Bounded Context)

도메인을 명확한 경계를 가진 하위 영역으로 나누는 개념이다. 같은 단어라도 컨텍스트마다 의미가 다를 수 있다.

예를 들어 "고객(Customer)"이라는 개념은:
- **주문 컨텍스트**: 주문자 정보, 배송지
- **마케팅 컨텍스트**: 구매 이력, 관심사, 세그먼트
- **결제 컨텍스트**: 결제 수단, 신용 한도

각 바운디드 컨텍스트 안에서는 유비쿼터스 언어가 일관되게 유지된다.

### 컨텍스트 맵 (Context Map)

바운디드 컨텍스트 사이의 관계를 나타낸 지도다. 주요 패턴:

| 패턴 | 설명 |
|------|------|
| 공유 커널 (Shared Kernel) | 두 팀이 도메인 모델 일부를 공유 |
| 고객-공급자 (Customer-Supplier) | 상류 팀이 하류 팀의 요구를 수용 |
| 순응주의자 (Conformist) | 하류 팀이 상류 모델을 그대로 따름 |
| 부패 방지 계층 (Anti-Corruption Layer) | 외부 시스템으로부터 내부 모델을 보호하는 변환 레이어 |
| 오픈 호스트 서비스 (Open Host Service) | 공개 프로토콜로 다른 컨텍스트에 서비스 제공 |

---

## 전술적 설계 (Tactical Design)

### 엔티티 (Entity)

고유한 식별자(ID)를 가지며, 생명주기 동안 상태가 변해도 동일 객체로 취급된다.

```java
public class Order {
    private final OrderId id;  // 식별자
    private OrderStatus status;
    private List<OrderLine> orderLines;

    // ID로 동등성 비교
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Order)) return false;
        Order order = (Order) o;
        return id.equals(order.id);
    }
}
```

### 값 객체 (Value Object)

식별자가 없고 속성 값으로 동등성을 판단한다. 불변(immutable)이어야 한다.

```java
public class Money {
    private final BigDecimal amount;
    private final Currency currency;

    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Currency mismatch");
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }

    // 속성 값으로 동등성 비교
    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Money)) return false;
        Money money = (Money) o;
        return amount.equals(money.amount) && currency.equals(money.currency);
    }
}
```

### 애그리거트 (Aggregate)

연관된 엔티티와 값 객체의 묶음으로, 하나의 **루트 엔티티(Aggregate Root)**를 통해서만 외부에서 접근할 수 있다.

```
Order (Aggregate Root)
├── OrderLine (Entity)
│   └── Money (Value Object)
├── ShippingAddress (Value Object)
└── OrderStatus (Value Object)
```

규칙:
- 외부에서는 반드시 애그리거트 루트를 통해서만 내부 객체에 접근
- 하나의 트랜잭션에서 하나의 애그리거트만 수정 (원칙)
- 애그리거트 경계는 일관성 경계이기도 함

```java
public class Order {  // Aggregate Root
    private List<OrderLine> orderLines;

    // 외부에서 직접 orderLine을 조작하지 않고 Order를 통해서만 접근
    public void addProduct(Product product, int quantity) {
        OrderLine line = new OrderLine(product.getId(), product.getPrice(), quantity);
        this.orderLines.add(line);
    }

    public Money totalAmount() {
        return orderLines.stream()
            .map(OrderLine::subtotal)
            .reduce(Money.ZERO, Money::add);
    }
}
```

### 도메인 서비스 (Domain Service)

특정 엔티티나 값 객체에 속하지 않는 도메인 로직을 처리한다. 상태를 갖지 않는다(stateless).

```java
public class TransferService {
    // 송금은 Account 하나에 속하지 않고 두 Account 사이의 연산
    public void transfer(Account from, Account to, Money amount) {
        from.withdraw(amount);
        to.deposit(amount);
    }
}
```

### 도메인 이벤트 (Domain Event)

도메인에서 발생한 중요한 사실을 나타낸다. 과거형으로 명명한다.

```java
public class OrderPlaced {
    private final OrderId orderId;
    private final CustomerId customerId;
    private final Money totalAmount;
    private final Instant occurredAt;
}
```

이벤트를 발행하면 다른 바운디드 컨텍스트가 이를 구독해서 처리할 수 있다. 예를 들어 `OrderPlaced` 이벤트가 발생하면:
- 결제 컨텍스트: 결제 처리 시작
- 재고 컨텍스트: 재고 차감
- 알림 컨텍스트: 주문 확인 이메일 발송

### 리포지토리 (Repository)

애그리거트의 영속성을 담당한다. 컬렉션처럼 동작하도록 인터페이스를 설계한다.

```java
public interface OrderRepository {
    Order findById(OrderId id);
    void save(Order order);
    void remove(Order order);
}

// 구현체는 인프라 레이어에 위치 (JPA, MyBatis 등)
public class JpaOrderRepository implements OrderRepository {
    @Override
    public Order findById(OrderId id) {
        return em.find(OrderEntity.class, id.value()).toDomain();
    }
}
```

---

## 레이어드 아키텍처와 DDD

DDD는 보통 다음 레이어 구조와 함께 사용한다.

```
Presentation Layer   ─── UI, API Controller
        │
Application Layer    ─── Use Case, Application Service (트랜잭션 경계)
        │
Domain Layer         ─── Entity, Value Object, Aggregate, Domain Service
        │
Infrastructure Layer ─── Repository 구현체, DB, 외부 API
```

- **도메인 레이어**는 다른 레이어에 의존하지 않는다
- **애플리케이션 서비스**는 도메인 객체를 조합해 유스케이스를 실현한다
- 인프라 의존성은 DIP(Dependency Inversion Principle)로 역전시킨다

```java
// Application Service 예시
public class PlaceOrderService {
    private final OrderRepository orderRepository;
    private final ProductRepository productRepository;

    public OrderId placeOrder(PlaceOrderCommand command) {
        Customer customer = customerRepository.findById(command.customerId());
        List<Product> products = productRepository.findByIds(command.productIds());

        Order order = Order.place(customer, products);
        orderRepository.save(order);

        return order.getId();
    }
}
```

---

## DDD가 적합한 상황

DDD는 복잡한 도메인 로직을 다룰 때 빛을 발한다. 단순 CRUD 시스템에는 오버엔지니어링이 될 수 있다.

| 적합한 경우 | 부적합한 경우 |
|-------------|---------------|
| 복잡한 비즈니스 규칙이 많을 때 | 단순 CRUD 위주 시스템 |
| 도메인 전문가와 지속적 협업이 필요할 때 | 빠른 프로토타이핑이 목적일 때 |
| 장기적으로 유지보수할 시스템 | 단기 프로젝트 |
| MSA 설계의 서비스 경계를 나눌 때 | 팀이 소규모이고 도메인이 단순할 때 |

---

## 정리

- **유비쿼터스 언어**: 팀 전체가 공유하는 도메인 언어, 코드에도 반영
- **바운디드 컨텍스트**: 도메인을 명확한 경계로 구분, 컨텍스트 내에서 언어 일관성 보장
- **엔티티**: ID로 동일성 판단, 가변 상태
- **값 객체**: 속성으로 동등성 판단, 불변
- **애그리거트**: 일관성 경계, 루트를 통해서만 외부 접근
- **도메인 서비스**: 여러 엔티티에 걸친 도메인 로직
- **도메인 이벤트**: 도메인에서 발생한 사실, 컨텍스트 간 통신에 활용
- **리포지토리**: 애그리거트 영속성, 컬렉션처럼 사용
