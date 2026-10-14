---
title: 비동기 생명주기 훅을 올바르게 사용한다
impact: HIGH
impactDescription: "초기화 경쟁 상태와 연결 누수, 시작 중 무한 대기를 방지함"
tags: performance, lifecycle, async, hooks
---

## 비동기 생명주기 훅을 올바르게 사용한다

요청을 받기 전에 필요한 작업은 Nest가 기다릴 수 있도록 Promise를 반환한다. 연결과 쿼리에 제한 시간을 설정하고, 초기화 실패를 전파해 준비되지 않은 인스턴스가 트래픽을 받지 않게 한다. 같은 모듈의 다른 프로바이더 훅이 먼저 끝난다고 가정하지 않는다. 생성자나 초기화 과정에서 필요한 의존성은 비동기 팩토리 프로바이더로 준비한다.

**잘못된 예 (초기화를 기다리지 않고 대여한 연결을 반납하지 않음):**

```typescript
@Injectable()
export class DatabaseService implements OnModuleInit {
  constructor(@Inject('PG_POOL') private readonly pool: Pool) {}

  onModuleInit(): void {
    this.pool.connect();
    // 연결 전에 서버가 시작되고 실패도 처리되지 않는다.
    // connect()는 연결을 대여하므로 성공해도 release() 없이는 풀이 고갈된다.
  }
}
```

**올바른 예 (준비를 기다리고 소유한 자원을 정리함):**

```typescript
import {
  Injectable, Logger, OnApplicationShutdown, OnModuleInit,
} from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { Pool } from 'pg';

// 이 프로바이더만 풀을 생성하고 종료한다. TypeORM 풀과 중복 생성하지 않는다.
@Injectable()
export class DatabaseService implements OnModuleInit, OnApplicationShutdown {
  private readonly logger = new Logger(DatabaseService.name);
  private readonly pool: Pool;
  private closing?: Promise<void>;

  constructor(config: ConfigService) {
    this.pool = new Pool({
      connectionString: config.getOrThrow<string>('DATABASE_URL'),
      connectionTimeoutMillis: 5_000,
      statement_timeout: 5_000,
      query_timeout: 5_000, // 연결 후 응답 대기도 클라이언트에서 제한한다
    });
    this.pool.on('error', () => {
      this.logger.error('Idle database connection failed');
    });
  }

  async onModuleInit(): Promise<void> {
    try {
      // 단일 쿼리는 자동으로 연결을 대여하고 반납한다.
      await this.pool.query('SELECT 1');
    } catch (error) {
      await this.closePool();
      throw error;
    }
  }

  async onApplicationShutdown(): Promise<void> {
    // HTTP 연결 종료 후 풀을 닫는다. 진행 중인 작업의 종료 순서도 조정한다.
    await this.closePool();
  }

  private closePool(): Promise<void> {
    return this.closing ??= this.pool.end();
  }
}

// 다른 프로바이더가 주입받는 시점부터 준비되어 있어야 하는 데이터의 예.
import { readFile } from 'node:fs/promises';
import { Module } from '@nestjs/common';

export const EMAIL_TEMPLATE = Symbol('EMAIL_TEMPLATE');

@Module({
  providers: [{
    provide: EMAIL_TEMPLATE,
    useFactory: async (): Promise<string> => {
      return readFile('templates/welcome.html', 'utf8');
    },
  }],
  exports: [EMAIL_TEMPLATE],
})
export class TemplatesModule {}
// 소비자는 @Inject(EMAIL_TEMPLATE)으로 주입한다.
// Nest는 팩토리 Promise가 완료된 뒤 그 의존성을 사용하는 프로바이더를 생성한다.
```

`onApplicationBootstrap()`은 모든 `onModuleInit()`이 끝난 뒤, 서버가 듣기 전에 실행된다. 필수 캐시 준비는 여기서 기다릴 수 있지만 선택적 워밍이 앱 가용성을 막지 않도록 실패 정책을 구분한다. 백그라운드 작업은 오류 처리와 종료 시 취소를 함께 구현한다. 요청 스코프와 지연 로딩 프로바이더에는 생명주기 훅이 호출되지 않는다.

신호 기반 종료를 사용하면 부트스트랩에서 `app.enableShutdownHooks()`를 호출한다. 수동 신호 처리기를 사용하는 앱은 [종료 규칙](devops-graceful-shutdown.md)의 방식으로 소유권을 하나로 정한다. `pool.connect()`가 필요한 트랜잭션은 `finally`에서 `client.release()`해야 한다.

검증: 초기화 거부 시 서버가 요청을 받지 않는지, 정상 시작 후 대여 연결이 남지 않는지, 종료 시 풀과 백그라운드 작업이 닫히는지 확인한다.

참고: [NestJS Lifecycle Events](https://docs.nestjs.com/fundamentals/lifecycle-events), [Async Providers](https://docs.nestjs.com/fundamentals/async-providers), [node-postgres Pool](https://node-postgres.com/apis/pool)
