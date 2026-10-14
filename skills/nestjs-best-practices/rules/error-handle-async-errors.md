---
title: 비동기 오류를 올바르게 처리한다
impact: HIGH
impactDescription: "처리되지 않은 Promise 거부와 조용히 유실되는 작업을 방지함"
tags: error-handling, async, promises
---

## 비동기 오류를 올바르게 처리한다

NestJS는 라우트 핸들러가 반환하거나 기다리는 Promise의 거부를 처리한다. 응답과 분리한 작업은 그 오류 처리 범위에서 벗어난다. 작업이 요청 성공의 일부라면 기다리고, 실패해도 요청에 영향을 주지 않는 작업만 명시적으로 오류를 기록하며 분리한다. 전달 보장이 필요하면 영속 큐와 트랜잭션 아웃박스를 사용한다.

**잘못된 예 (Promise를 관찰하지 않음):**

```typescript
@Injectable()
export class UsersService {
  async createUser(dto: CreateUserDto): Promise<User> {
    const user = await this.repo.save(dto);
    this.emailService.sendWelcome(user.email); // 거부가 요청의 예외 처리 범위 밖에 있다.
    return user;
  }
}

@OnEvent('order.created')
handleOrderCreated(event: OrderCreatedEvent) {
  this.processOrder(event); // 반환하거나 await하지 않아 리스너도 완료·실패를 관찰하지 못한다.
}
```

**올바른 예 (작업의 실패 계약을 정하고 Promise를 관찰함):**

```typescript
// 반드시 전달할 필요가 없는 알림에만 사용한다.
@Injectable()
export class WelcomeNotifier {
  private readonly logger = new Logger(WelcomeNotifier.name);

  constructor(private readonly emailService: EmailService) {}

  async notify(email: string): Promise<void> {
    try {
      await this.emailService.sendWelcome(email);
    } catch (error: unknown) {
      this.logger.error(
        'Failed to send welcome email',
        error instanceof Error ? error.stack : 'Non-Error rejection',
      );
    }
  }
}

// 요청과 독립된 최선형 알림: 프로세스 종료 시 유실될 수 있다.
void this.welcomeNotifier.notify(user.email);

// 프로세스 내 이벤트의 실패를 호출자에게 전달해야 하는 경우
@Injectable()
export class OrderListener {
  constructor(private readonly orderProcessor: OrderProcessor) {}

  @OnEvent('order.created', { suppressErrors: false })
  async handleOrderCreated(event: OrderCreatedEvent): Promise<void> {
    await this.orderProcessor.process(event);
  }
}

@Injectable()
export class OrderNotifier {
  constructor(private readonly eventEmitter: EventEmitter2) {}

  async notify(event: OrderCreatedEvent): Promise<void> {
    // emit()은 리스너의 Promise를 기다리지 않는다.
    // emitAsync()와 suppressErrors: false를 함께 써야 호출자가 실패를 관찰한다.
    await this.eventEmitter.emitAsync('order.created', event);
  }
}

@Injectable()
export class CleanupJob {
  constructor(private readonly cleanupService: CleanupService) {}

  @Cron('0 0 * * *')
  async dailyCleanup(): Promise<void> {
    await this.cleanupService.run();
    // Nest 스케줄러가 예외를 기록한다. 재시도·알림은 별도 정책으로 구성한다.
  }
}

// main.ts: 애플리케이션 생성 전에 치명적 오류 관찰자를 등록한다.
import { writeSync } from 'node:fs';

process.on('uncaughtExceptionMonitor', (error, origin) => {
  // 프로세스가 종료될 때 비동기 로그가 비워진다고 가정하지 않는다.
  writeSync(process.stderr.fd, `${origin}: ${error.stack ?? error.message}\n`);
});

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
}

void bootstrap().catch((error: unknown) => {
  writeSync(process.stderr.fd, `${error instanceof Error ? error.stack : 'Bootstrap failed'}\n`);
  process.exit(1); // 초기화 도중 남은 핸들이 있어도 실패한 프로세스를 유지하지 않는다.
});
```

`EventEmitterModule.forRoot()`와 `ScheduleModule.forRoot()` 및 각 리스너·잡 프로바이더를 등록한다. `OrderNotifier.notify()`를 호출하는 쪽도 Promise를 반환하거나 await해야 한다. `emitAsync()`는 트랜잭션이나 영속 큐가 아니므로 한 리스너의 실패가 이미 완료된 다른 리스너의 변경을 되돌리지 않는다. 재고·결제 같은 작업은 [이벤트 규칙](arch-use-events.md)의 정합성·재시도 계약을 적용한다.

`uncaughtExceptionMonitor`는 관찰자이며 복구 수단이 아니다. `uncaughtException`/`unhandledRejection` 리스너로 기록만 하고 프로세스를 계속 살려두지 않는다. Node의 처리되지 않은 Promise 거부 정책은 실행 옵션으로 확인하고 치명적 오류 후 재시작은 운영 환경에서 관리한다. `void`는 린터에 의도를 표시할 뿐 Promise 오류를 처리하지 않는다.

참고: [NestJS Events](https://docs.nestjs.com/techniques/events), [NestJS Task Scheduling](https://docs.nestjs.com/techniques/task-scheduling), [Node.js uncaughtExceptionMonitor](https://nodejs.org/api/process.html#event-uncaughtexceptionmonitor)
