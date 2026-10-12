---
title: 메시지 패턴과 이벤트 패턴을 올바르게 사용한다
impact: MEDIUM
impactDescription: "올바른 패턴으로 안정적인 마이크로서비스 통신을 보장함"
tags: microservices, message-pattern, event-pattern, communication
---

## 메시지 패턴과 이벤트 패턴을 올바르게 사용한다

NestJS 마이크로서비스는 요청과 응답을 주고받는 MessagePattern, 이벤트를 전달하는 EventPattern이라는 두 가지 통신 패턴을 지원한다. 응답이 필요하면 MessagePattern을, 결과를 기다리지 않는 알림에는 EventPattern을 사용한다. 둘의 차이를 이해하면 통신 버그를 방지할 수 있다.

**잘못된 예 (사용 사례에 맞지 않는 패턴을 사용함):**

```typescript
// 응답을 기다리지 않는 작업에 @MessagePattern 사용
@Controller()
export class NotificationsController {
  @MessagePattern('user.created')
  async handleUserCreated(data: UserCreatedEvent) {
    // 응답을 기다리므로 발신자의 진행을 막음
    await this.emailService.sendWelcome(data.email);
    // 이메일 전송이 실패하면 발신자가 오류를 받음 (결합 발생!)
  }
}

// 응답을 기대하면서 @EventPattern 사용
@Controller()
export class OrdersController {
  @EventPattern('inventory.check')
  async checkInventory(data: CheckInventoryDto) {
    const available = await this.inventory.check(data);
    return available; // @EventPattern에서는 이 반환값을 무시함!
  }
}

// 클라이언트의 강한 결합
@Injectable()
export class UsersService {
  async createUser(dto: CreateUserDto): Promise<User> {
    const user = await this.repo.save(dto);

    // 알림 서비스가 응답할 때까지 대기
    await this.client.send('user.created', user).toPromise();
    // 알림 서비스가 중단되면 사용자 생성이 실패함!

    return user;
  }
}
```

**올바른 예 (요청과 응답에는 MessagePattern을, 결과를 기다리지 않는 전달에는 EventPattern을 사용함):**

```typescript
// MessagePattern: 요청과 응답 (응답이 필요한 경우)
@Controller()
export class InventoryController {
  @MessagePattern({ cmd: 'check_inventory' })
  async checkInventory(data: CheckInventoryDto): Promise<InventoryResult> {
    const result = await this.inventoryService.check(data.productId, data.quantity);
    return result; // 호출자에게 응답 전달
  }
}

// 클라이언트가 응답을 기대함
@Injectable()
export class OrdersService {
  async createOrder(dto: CreateOrderDto): Promise<Order> {
    // 재고 확인: 계속 진행하려면 이 응답이 필요함
    const inventory = await firstValueFrom(
      this.inventoryClient.send<InventoryResult>(
        { cmd: 'check_inventory' },
        { productId: dto.productId, quantity: dto.quantity },
      ),
    );

    if (!inventory.available) {
      throw new BadRequestException('Insufficient inventory');
    }

    return this.repo.save(dto);
  }
}

// EventPattern: 결과를 기다리지 않는 전달 (알림과 부수 효과)
@Controller()
export class NotificationsController {
  @EventPattern('user.created')
  async handleUserCreated(data: UserCreatedEvent): Promise<void> {
    // 반환값 없이 이벤트만 처리
    await this.emailService.sendWelcome(data.email);
    await this.analyticsService.track('user_signup', data);
    // 실패해도 발신자에게 영향을 주지 않음
  }
}

// 클라이언트가 기다리지 않고 이벤트 발행
@Injectable()
export class UsersService {
  async createUser(dto: CreateUserDto): Promise<User> {
    const user = await this.repo.save(dto);

    // 실행 후 기다리지 않으며 진행을 막지 않음
    this.eventClient.emit('user.created', {
      userId: user.id,
      email: user.email,
      timestamp: new Date(),
    });

    return user; // 이벤트 처리와 관계없이 사용자 생성 성공
  }
}

// 중요 이벤트를 위한 혼합 패턴
@Injectable()
export class OrdersService {
  async createOrder(dto: CreateOrderDto): Promise<Order> {
    const order = await this.repo.save(dto);

    // 중요 작업: 재고 예약 (MessagePattern 사용)
    const reserved = await firstValueFrom(
      this.inventoryClient.send({ cmd: 'reserve_inventory' }, {
        orderId: order.id,
        items: dto.items,
      }),
    );

    if (!reserved.success) {
      await this.repo.delete(order.id);
      throw new BadRequestException('Could not reserve inventory');
    }

    // 비핵심 작업: 알림 (EventPattern 사용)
    this.eventClient.emit('order.created', {
      orderId: order.id,
      userId: dto.userId,
      total: dto.total,
    });

    return order;
  }
}

// 오류 처리 패턴
// MessagePattern 오류는 호출자에게 전파됨
@MessagePattern({ cmd: 'get_user' })
async getUser(userId: string): Promise<User> {
  const user = await this.repo.findOne({ where: { id: userId } });
  if (!user) {
    throw new RpcException('User not found'); // 호출자가 수신함
  }
  return user;
}

// EventPattern 오류는 처리하는 쪽에서 다뤄야 함
@EventPattern('order.created')
async handleOrderCreated(data: OrderCreatedEvent): Promise<void> {
  try {
    await this.processOrder(data);
  } catch (error) {
    // 로그를 기록하고 필요하면 재시도하며 예외는 던지지 않음
    this.logger.error('Failed to process order event', error);
    await this.deadLetterQueue.add(data);
  }
}
```

참고: [NestJS Microservices](https://docs.nestjs.com/microservices/basics)
