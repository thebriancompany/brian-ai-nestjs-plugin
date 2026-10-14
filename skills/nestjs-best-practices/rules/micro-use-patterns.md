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
  async checkAvailability(dto: CheckInventoryDto): Promise<InventoryResult> {
    // 조회 시점의 가용 재고를 반환하며, 재고를 예약하지는 않음
    return firstValueFrom(
      this.inventoryClient.send<InventoryResult>(
        { cmd: 'check_inventory' },
        { productId: dto.productId, quantity: dto.quantity },
      ).pipe(timeout({ first: 3000 })),
    );
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
    // 처리 실패는 발신자의 응답으로 전달되지 않는다. 소비 측 재시도 정책이 필요하다.
  }
}

// 클라이언트가 기다리지 않고 이벤트 발행
@Injectable()
export class UsersService {
  private readonly logger = new Logger(UsersService.name);

  async createUser(dto: CreateUserDto): Promise<User> {
    const user = await this.repo.save(dto);

    // emit()은 hot Observable이다. 전송 오류를 관찰하되 소비자 완료 응답으로 간주하지 않는다.
    this.eventClient.emit('user.created', {
      userId: user.id,
      email: user.email,
      timestamp: new Date().toISOString(),
    }).subscribe({
      error: () => this.logger.error('user.created publish failed'),
    });

    return user; // 이벤트 처리와 관계없이 사용자 생성 성공
  }
}

// 오류 처리 패턴
// MessagePattern 오류는 호출자에게 전파됨
@MessagePattern({ cmd: 'get_user' })
async getUser(userId: string): Promise<UserResponseDto> {
  const user = await this.repo.findOne({ where: { id: userId } });
  if (!user) {
    throw new RpcException('User not found'); // 호출자가 수신함
  }
  return { id: user.id, name: user.name };
}

// EventPattern의 재시도·ack는 전송 계층 설정과 함께 결정한다.
@EventPattern('order.created')
async handleOrderCreated(data: OrderCreatedEvent): Promise<void> {
  await this.processOrder(data); // eventId를 기준으로 멱등 처리하고 실패를 전파한다.
  // 오류를 삼켜 성공으로 ack하지 않는다. RPC 예외 자체가 재배달을 보장하지는 않는다.
}
```

클라이언트와 서버의 패턴 이름·payload DTO를 일치시키고 메시지 입력도 검증한다. `timeout`과 `firstValueFrom`은 RxJS에서 가져온다. 앱 종료 시 `ClientProxy.close()`로 연결을 정리한다. RabbitMQ 수동 ack, Kafka offset 등 전달 확인 방식은 다르므로 실패 재시도·최대 횟수·dead letter 처리를 사용하는 전송 계층에 맞게 구성한다. 이메일과 분석 이벤트가 함께 처리되면 후속 실패로 이메일이 중복 전송될 수 있으므로 각각 멱등 처리하거나 작업을 분리한다.

### 주문과 재고처럼 여러 서비스의 상태를 변경하는 경우

MessagePattern은 요청과 응답을 연결하지만 주문 저장과 원격 재고 예약을 하나의 트랜잭션으로 만들지는 않는다. 예약이 커밋된 뒤 응답이 유실되거나 주문 서비스가 종료될 수 있다. 타임아웃이나 통신 오류만으로 예약 실패를 확정하거나 로컬 주문을 삭제하면 안 된다. 위의 재고 조회 결과도 이후 주문의 재고 확보를 보장하지 않는다.

중요한 작업에는 saga나 영속적인 작업 상태를 사용하는 복구 흐름을 설계한다.

- 주문을 `pending` 상태로 저장하고, 재시작 후에도 처리할 작업을 찾을 수 있게 한다. DB 저장과 메시지 발행을 함께 보장해야 한다면 주문과 outbox 레코드를 같은 로컬 트랜잭션으로 기록하고 별도 전달자가 재시도한다.
- 재고 예약에는 재시도해도 바뀌지 않는 멱등 키를 사용한다. 재고 서비스는 예약 변경과 해당 키의 처리 결과를 원자적으로 저장하고, 동일 요청에는 기존 결과를 반환한다. 주문 생성 요청 자체의 재시도도 같은 주문으로 연결해야 한다.
- 응답이 없으면 결과를 조회하거나 같은 키로 재시도한다. 확인된 결과에 따라 주문을 `confirmed` 또는 `cancelled`로 전이하며, 일시적인 통신 실패는 재처리할 상태로 남긴다.
- 예약을 취소해야 한다면 예약 해제도 멱등하게 수행하고, 보상 진행 상태를 저장해 실패하거나 재시작해도 이어서 처리한다. 늦게 도착한 예약 요청·응답이 취소를 되돌리지 않도록 상태 전이를 제어한다.

앞의 `emit()` 예제는 전달 누락을 허용하는 알림에 한정한다. 주문 확정 이벤트처럼 전달이 필수인 이벤트도 상태 변경과 outbox 기록을 같은 트랜잭션으로 저장하고 소비자를 멱등하게 만든다. Outbox만으로 원격 작업의 성공이나 보상 완료까지 보장되지는 않는다. 관련 내용은 [트랜잭션 관리](db-use-transactions.md)와 [이벤트 사용](arch-use-events.md)을 참고한다.

참고: [NestJS Microservices](https://docs.nestjs.com/microservices/basics), [Transactional outbox](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html), [Saga orchestration](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/saga-orchestration.html)
