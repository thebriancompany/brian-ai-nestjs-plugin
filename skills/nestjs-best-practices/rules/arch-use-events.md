---
title: 이벤트 기반 아키텍처로 결합도를 낮춘다
impact: MEDIUM-HIGH
impactDescription: "비동기 처리와 모듈화를 가능하게 함"
tags: architecture, events, decoupling
---

## 이벤트 기반 아키텍처로 결합도를 낮춘다

같은 프로세스 안의 알림에는 `@nestjs/event-emitter`를 사용할 수 있다. 이벤트는 호출자와 구독자의 직접 결합을 줄이지만, 자동으로 백그라운드 작업이나 영속 전달이 되지는 않는다. 다른 프로세스에 전달하거나 재시도·유실 방지가 필요하면 메시지 브로커와 전달 보장 설계를 함께 선택한다. 필수 업무의 성공 여부를 호출자가 알아야 한다면 직접 호출을 유지해도 된다.

**개선 대상 (필수 작업과 비핵심 부수 작업을 한 흐름에 묶음):**

```typescript
// 서비스 간 직접 결합
@Injectable()
export class OrdersService {
  constructor(
    private inventoryService: InventoryService,
    private emailService: EmailService,
    private analyticsService: AnalyticsService,
    private notificationService: NotificationService,
    private loyaltyService: LoyaltyService,
  ) {}

  async createOrder(dto: CreateOrderDto): Promise<Order> {
    const order = await this.repo.save(dto);

    // 강한 결합: OrdersService가 모든 사용처를 알고 있음
    await this.inventoryService.reserve(order.items);
    await this.emailService.sendConfirmation(order);
    await this.analyticsService.track('order_created', order);
    await this.notificationService.push(order.userId, 'Order placed');
    await this.loyaltyService.addPoints(order.userId, order.total);

    // 새 동작을 추가하려면 이 서비스를 수정해야 함
    return order;
  }
}
```

**올바른 예 (필수 작업은 트랜잭션으로 묶고 비핵심 알림만 이벤트로 분리함):**

```typescript
import { EventEmitter2, OnEvent } from '@nestjs/event-emitter';

export class OrderCreatedEvent {
  constructor(public readonly orderId: string, public readonly total: number) {}
}

@Injectable()
export class OrdersService {
  private readonly logger = new Logger(OrdersService.name);

  constructor(
    private dataSource: DataSource,
    private inventoryService: InventoryService,
    private pricingService: PricingService,
    private eventEmitter: EventEmitter2,
  ) {}

  async createOrder(userId: string, dto: CreateOrderDto): Promise<Order> {
    const order = await this.dataSource.transaction(async manager => {
      // DTO의 상품 ID·수량을 검증하고 서버의 상품 가격으로 합계를 계산한다.
      const pricedOrder = await this.pricingService.quote(manager, dto.items);
      // reserve는 같은 manager로 재고 존재·수량을 검증하고 원자적으로 차감한다.
      // 외부 서비스 호출이라면 이 트랜잭션에 포함되지 않으므로 별도 상태/보상이 필요하다.
      await this.inventoryService.reserve(manager, dto.items);
      return manager.save(Order, {
        userId, // 인증된 호출자에게서 전달받으며 DTO로 덮어쓰지 않는다.
        items: pricedOrder.items,
        total: pricedOrder.total,
        status: 'confirmed',
      });
    });

    // 유실을 허용하는 비핵심 통계 알림만 메모리 이벤트로 보낸다.
    // 실패한 알림 때문에 이미 커밋된 주문을 실패로 응답하지 않는다.
    try {
      void this.eventEmitter.emitAsync(
        'order.created', new OrderCreatedEvent(order.id, order.total),
      ).catch(() => this.logger.warn('Order analytics event failed'));
    } catch {
      this.logger.warn('Order analytics event failed');
    }
    return order;
  }
}

@Injectable()
export class AnalyticsListener {
  constructor(private analyticsService: AnalyticsService) {}

  @OnEvent('order.created', { suppressErrors: false })
  async handleOrderCreated(event: OrderCreatedEvent): Promise<void> {
    await this.analyticsService.track('order_created', {
      orderId: event.orderId, total: event.total,
    });
  }
}
```

위 예제는 이벤트 경계를 보여주는 발췌다. 주문 항목을 함께 저장하는 엔티티 매핑 또는 명시적인 항목 저장이 필요하며, `PricingService`와 `InventoryService`의 DB 연산은 전달받은 `manager`를 사용해야 한다. `EventEmitterModule.forRoot()`를 루트 모듈에서 한 번 등록하고 `AnalyticsListener`를 기능 모듈의 `providers`에 등록한다.

`emit()`은 비동기 리스너 완료를 기다리지 않는다. 완료·실패를 관찰하려면 `emitAsync()`와 오류 전파 설정을 함께 사용한다. 이 예제의 `void ...catch(...)`는 실패를 기록하지만 요청·종료 시 완료를 보장하지 않으므로 유실 가능한 통계에만 사용한다. 시작 시 이벤트를 발행한다면 리스너 등록 완료도 확인해야 한다.

이메일·결제·재고처럼 전달 누락을 허용할 수 없는 작업은 DB 변경과 outbox 기록을 같은 트랜잭션으로 저장하고, 별도 전달자가 재시도하며 소비자는 멱등하게 처리한다. outbox도 외부 결제를 DB 트랜잭션에 포함시키지는 않는다. 외부 작업에는 `pending` 등 상태 전이와 실패 후 보상을 설계한다. 결합도를 낮추기 위해 성공 조건이나 오류 전파 계약을 바꾸지 않는다.

참고: [NestJS Events](https://docs.nestjs.com/techniques/events)
