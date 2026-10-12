---
title: 비동기 오류를 올바르게 처리한다
impact: HIGH
impactDescription: "처리되지 않은 Promise 거부로 인한 프로세스 장애를 방지함"
tags: error-handling, async, promises
---

## 비동기 오류를 올바르게 처리한다

NestJS는 비동기 라우트 핸들러에서 발생하는 오류를 자동으로 처리하지만, 백그라운드 작업, 이벤트 핸들러, 직접 만든 Promise의 오류는 애플리케이션을 비정상 종료시킬 수 있다. 항상 비동기 오류를 명시적으로 처리하고 전역 핸들러를 최후의 방어 수단으로 사용한다.

**잘못된 예 (오류 처리 없이 실행하고 기다리지 않음):**

```typescript
// 오류 처리 없이 실행하고 기다리지 않음
@Injectable()
export class UsersService {
  async createUser(dto: CreateUserDto): Promise<User> {
    const user = await this.repo.save(dto);

    // 실행하고 기다리지 않으므로 실패 시 오류를 처리하지 못함!
    this.emailService.sendWelcome(user.email);

    return user;
  }
}

// 이벤트 핸들러에서 처리하지 않은 Promise
@Injectable()
export class OrdersService {
  @OnEvent('order.created')
  handleOrderCreated(event: OrderCreatedEvent) {
    // Promise를 반환하거나 await하지 않아 리스너의 오류 처리 범위에서 벗어남
    this.processOrder(event);
    // 내부에서 거부된 Promise가 처리되지 않을 수 있음
  }

  private async processOrder(event: OrderCreatedEvent): Promise<void> {
    await this.inventoryService.reserve(event.items);
    await this.notificationService.send(event.userId);
  }
}

// 예약 작업에 업무 복구 정책이 없음
@Cron('0 0 * * *')
async dailyCleanup(): Promise<void> {
  await this.cleanupService.run();
  // Nest 스케줄러가 예외를 기록하지만 재시도·복구 정책을 대신 구현하지는 않음
}
```

**올바른 예 (비동기 오류를 명시적으로 처리함):**

```typescript
// 기다리지 않는 작업도 명시적인 catch로 오류 처리
@Injectable()
export class UsersService {
  private readonly logger = new Logger(UsersService.name);

  async createUser(dto: CreateUserDto): Promise<User> {
    const user = await this.repo.save(dto);

    // 오류를 명시적으로 잡아 로그 기록
    this.emailService.sendWelcome(user.email).catch((error) => {
      this.logger.error('Failed to send welcome email', error.stack);
      // 필요하면 재시도 큐에 추가
    });

    return user;
  }
}

// 비동기 이벤트 핸들러를 올바르게 처리
@Injectable()
export class OrdersService {
  private readonly logger = new Logger(OrdersService.name);

  @OnEvent('order.created')
  async handleOrderCreated(event: OrderCreatedEvent): Promise<void> {
    try {
      await this.processOrder(event);
    } catch (error) {
      this.logger.error('Failed to process order', { event, error });
      // 재처리 큐로 전달한다. 큐 발행 실패도 모니터링해야 한다.
      await this.deadLetterQueue.add('order.created', event);
    }
  }
}

// 안전한 예약 작업
@Injectable()
export class CleanupService {
  private readonly logger = new Logger(CleanupService.name);

  @Cron('0 0 * * *')
  async dailyCleanup(): Promise<void> {
    try {
      await this.cleanupService.run();
      this.logger.log('Daily cleanup completed');
    } catch (error) {
      this.logger.error('Daily cleanup failed', error.stack);
      // 알림 또는 재시도 로직
    }
  }
}

// main.ts: 치명적 오류의 기본 종료 동작을 바꾸지 않고 관찰한다.
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  const logger = new Logger('Bootstrap');

  process.on('uncaughtExceptionMonitor', (error) => {
    logger.error('Uncaught Exception:', error);
  });

  await app.listen(3000);
}
```

전역 관찰자는 복구 수단이 아니다. Node의 처리되지 않은 Promise 거부 정책은 실행 옵션으로 확인하고, 치명적 오류 후 프로세스 재시작은 운영 환경에서 관리한다. 반드시 전달해야 하는 이메일·재고 작업은 `.catch()`로 기록하는 것만으로 보장되지 않는다. [이벤트 규칙](arch-use-events.md)의 트랜잭션·아웃박스와 재시도 계약을 적용한다.

참고: [Node.js Unhandled Rejections](https://nodejs.org/api/process.html#event-unhandledrejection)
