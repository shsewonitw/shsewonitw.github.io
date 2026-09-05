---
layout: post
title: "[Daily morning study] 서비스 디스커버리(Service Discovery) 패턴과 구현 방법"
description: >
  #daily morning study
category: 
    - dms
    - dms-backend
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 왜 서비스 디스커버리가 필요한가

모놀리스 시대에는 서비스 주소가 고정 IP 하나였다. 그냥 nginx 설정에 적어두면 그만이었다.

MSA로 넘어오면 상황이 달라진다. 수십 개의 마이크로서비스가 각자 여러 인스턴스를 띄우고, 오토스케일링으로 수시로 늘었다 줄었다 한다. 컨테이너 오케스트레이터가 파드를 재스케줄링하면 IP가 바뀐다. 배포가 일어날 때마다 IP가 교체된다.

이 상황에서 서비스 A가 서비스 B를 호출하려면 어떻게 해야 할까? B의 현재 주소를 어딘가에서 동적으로 알아내야 한다. 이게 **서비스 디스커버리**다.

---

## 핵심 구성 요소: 서비스 레지스트리

서비스 디스커버리의 핵심은 **서비스 레지스트리(Service Registry)**다. 현재 살아있는 서비스 인스턴스의 주소(IP + 포트)를 중앙에서 관리하는 데이터베이스다.

인스턴스가 올라오면 자신을 레지스트리에 등록(Register)하고, 내려가면 등록을 해제(Deregister)한다. 레지스트리는 지속적으로 헬스 체크를 수행해서 죽은 인스턴스를 자동으로 제거한다.

```
[Service Instance A] ──등록──▶ [Service Registry]
[Service Instance B] ──등록──▶ (IP:Port 목록 보관)
[Caller]             ──조회──▶ [Service Registry] ──▶ 사용 가능한 인스턴스 반환
```

---

## 두 가지 디스커버리 패턴

### 클라이언트 사이드 디스커버리 (Client-Side Discovery)

호출하는 쪽(클라이언트)이 직접 레지스트리에 물어보고, 받은 목록에서 인스턴스를 골라 직접 호출한다.

```
Client ──1. 쿼리──▶ Service Registry
       ◀──2. 목록── (인스턴스 A, B, C)
       ──3. 직접 호출──▶ Instance B (클라이언트가 로드밸런싱)
```

**장점**
- 중간 프록시가 없어서 네트워크 홉이 적다.
- 클라이언트가 로드밸런싱 전략을 직접 제어할 수 있다 (가중치, zone-aware 등).

**단점**
- 모든 클라이언트가 레지스트리 클라이언트 라이브러리를 내장해야 한다.
- 언어/프레임워크마다 라이브러리를 별도 관리해야 해서 다언어 환경에서 복잡해진다.

대표 구현체: **Netflix Eureka + Ribbon** 조합.

---

### 서버 사이드 디스커버리 (Server-Side Discovery)

클라이언트는 로드밸런서(또는 API 게이트웨이)에만 요청을 보낸다. 로드밸런서가 레지스트리에 물어보고 적절한 인스턴스로 라우팅해준다.

```
Client ──요청──▶ Load Balancer ──1. 쿼리──▶ Service Registry
                               ◀──2. 목록──
                               ──3. 라우팅──▶ Instance A
```

**장점**
- 클라이언트는 디스커버리 로직을 전혀 모른다. 단순하다.
- 언어, 프레임워크에 무관하게 동일하게 작동한다.

**단점**
- 로드밸런서가 단일 장애 지점(SPOF)이 될 수 있다 (HA 구성 필요).
- 네트워크 홉이 하나 더 생긴다.

대표 구현체: **AWS ALB + ECS**, **Kubernetes Service + kube-proxy**.

---

## 등록 방식: Self-Registration vs Third-Party Registration

### Self-Registration

인스턴스가 시작할 때 직접 레지스트리에 등록하고, 종료될 때 직접 해제한다.

```python
# 예: FastAPI + Consul self-registration 패턴
import consul
import atexit

c = consul.Consul(host="consul", port=8500)

def register_service():
    c.agent.service.register(
        name="order-service",
        service_id="order-service-1",
        address="10.0.1.5",
        port=8080,
        check=consul.Check.http(
            url="http://10.0.1.5:8080/health",
            interval="10s",
            deregister="30s"
        )
    )

def deregister_service():
    c.agent.service.deregister("order-service-1")

register_service()
atexit.register(deregister_service)
```

**장점**: 구현이 간단하다.  
**단점**: 서비스가 레지스트리를 알아야 한다. 서비스와 레지스트리가 결합된다.

---

### Third-Party Registration

서비스 자신이 등록하지 않는다. 외부 시스템(배포 플랫폼, 오케스트레이터)이 서비스를 감지해서 레지스트리에 등록/해제한다.

Kubernetes가 대표적이다. 파드가 뜨고 내려가는 걸 kube-controller-manager가 감지해서 Endpoints 오브젝트를 갱신한다. 서비스 코드는 등록 로직을 전혀 모른다.

---

## 헬스 체크

레지스트리가 죽은 인스턴스를 걸러내려면 헬스 체크가 필수다. 주요 방식은 세 가지다.

| 방식 | 설명 | 예시 |
|------|------|------|
| HTTP 헬스 체크 | 레지스트리가 주기적으로 `/health` 엔드포인트 호출 | Consul HTTP check |
| TTL 기반 | 서비스가 주기적으로 "나 살아있음" 신호 전송, 신호 없으면 죽은 것으로 간주 | Consul TTL check |
| TCP 연결 | 포트에 TCP 연결이 가능한지 확인 | 간단하지만 애플리케이션 레벨 오류는 감지 못함 |

헬스 체크 엔드포인트는 단순히 `200 OK`만 리턴하면 안 된다. DB 연결, 외부 의존성 등 실질적인 동작 가능 여부를 확인해야 한다.

```python
@app.get("/health")
async def health_check():
    # DB 연결 확인
    try:
        await db.execute("SELECT 1")
    except Exception:
        raise HTTPException(status_code=503, detail="DB unavailable")
    
    # 캐시 연결 확인
    if not await redis.ping():
        raise HTTPException(status_code=503, detail="Cache unavailable")
    
    return {"status": "ok"}
```

---

## 주요 구현체 비교

| | Consul | Eureka | Kubernetes DNS |
|--|--------|--------|----------------|
| 개발사 | HashiCorp | Netflix | CNCF |
| 등록 방식 | Self / Agent | Self | Third-party (k8s) |
| 헬스 체크 | HTTP, TCP, TTL, Script | Client heartbeat | Readiness probe |
| DNS 지원 | ✅ | ❌ (별도 라이브러리) | ✅ |
| 다중 데이터센터 | ✅ | ❌ | 제한적 |
| 언어 독립성 | ✅ | JVM 편향 | ✅ |

---

## Kubernetes에서의 서비스 디스커버리

쿠버네티스는 서비스 디스커버리를 플랫폼 수준에서 내장하고 있다.

```yaml
# Service 오브젝트가 파드 셀렉터 기반으로 엔드포인트를 자동 관리
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order          # 이 레이블을 가진 파드들을 엔드포인트로 등록
  ports:
    - port: 80
      targetPort: 8080
```

클러스터 내 다른 파드에서는 `http://order-service` 또는 `http://order-service.default.svc.cluster.local`로 접근하면 kube-proxy가 살아있는 파드로 라우팅해준다.

DNS 조회 흐름:

```
order-service.default.svc.cluster.local
           ↓ CoreDNS 조회
    ClusterIP: 10.96.50.100
           ↓ kube-proxy (iptables / IPVS 규칙)
    실제 파드 IP (10.244.1.3, 10.244.2.7, ...)
```

파드가 추가/제거되면 쿠버네티스 컨트롤 플레인이 Endpoints 오브젝트를 갱신하고, kube-proxy가 iptables 규칙을 자동으로 업데이트한다.

---

## 패턴 선택 기준

- **쿠버네티스 환경**: 별도 레지스트리 없이 쿠버네티스 Service + CoreDNS만으로 충분한 경우가 대부분이다.
- **다중 플랫폼 / VM + 컨테이너 혼재**: Consul이 적합하다. VM 에이전트와 컨테이너 환경 모두 지원한다.
- **Spring 기반 JVM 서비스**: Eureka + Spring Cloud가 생태계 통합이 잘 되어 있다.
- **클라이언트 라이브러리를 피하고 싶다**: 서버 사이드 디스커버리(API 게이트웨이 + 레지스트리) 조합으로 클라이언트를 단순하게 유지한다.

---

## 정리

서비스 디스커버리는 MSA에서 서비스 간 통신의 기반 인프라다. 어떤 패턴을 선택하든 세 가지 문제를 해결해야 한다.

1. **등록**: 서비스가 어떻게 레지스트리에 자신을 알리는가
2. **헬스 체크**: 죽은 인스턴스를 어떻게 제거하는가
3. **조회와 라우팅**: 클라이언트가 어디서 어떻게 주소를 찾는가

쿠버네티스를 쓴다면 이 세 가지를 플랫폼이 대신 처리해준다. 그렇지 않은 환경이라면 Consul 같은 전용 솔루션으로 명시적으로 구성해야 한다.
