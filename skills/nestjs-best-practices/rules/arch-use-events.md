---
title: 이벤트 기반 아키텍처로 결합도를 낮춘다
impact: MEDIUM-HIGH
impactDescription: "비동기 처리와 모듈화를 가능하게 함"
tags: architecture, events, decoupling
---

## 이벤트 기반 아키텍처로 결합도를 낮춘다

서비스 내부 이벤트에는 `@nestjs/event-emitter`를, 서비스 간 통신에는 메시지 브로커를 사용한다. 이벤트를 사용하면 모듈이 직접 의존하지 않고도 변경에 반응할 수 있어 모듈화가 개선되고 비동기 처리가 가능해진다.

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
    private eventEmitter: EventEmitter2,
  ) {}

  async createOrder(dto: CreateOrderDto): Promise<Order> {
    const order = await this.dataSource.transaction(async manager => {
      // reserve는 같은 manager로 재고 존재·수량을 검증하고 원자적으로 차감한다.
      // 외부 서비스 호출이라면 이 트랜잭션에 포함되지 않으므로 별도 상태/보상이 필요하다.
      await this.inventoryService.reserve(manager, dto.items);
      return manager.save(Order, { ...dto, status: 'confirmed' });
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

이메일·결제·재고처럼 전달 누락을 허용할 수 없는 작업은 DB 변경과 outbox 기록을 같은 트랜잭션으로 저장하고, 별도 전달자가 재시도하며 소비자는 멱등하게 처리한다. `EventEmitterModule.forRoot()`와 리스너를 등록해야 한다. 단순한 서비스 호출도 적절한 설계이며, 결합도를 낮추기 위해 성공 조건이나 오류 전파 계약을 바꾸지 않는다.

참고: [NestJS Events](https://docs.nestjs.com/techniques/events)
