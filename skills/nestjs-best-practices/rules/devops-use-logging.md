---
title: 구조화된 로깅을 사용한다
impact: MEDIUM-HIGH
impactDescription: "요청 문맥을 안전하게 기록해 장애와 감사 이벤트를 추적함"
tags: devops, logging, structured-logs, cls
---

## 구조화된 로깅을 사용한다

JSON 로그에 이벤트 이름과 요청 문맥을 담고 수준을 구분한다. 비밀번호·토큰·쿠키·전체 DTO를 기록하지 않는다. 로그의 사용자 ID는 인증된 주체에서 가져온다. 쿼리 문자열에는 토큰이나 개인정보가 들어갈 수 있으므로 전체 URL도 기본적으로 기록하지 않는다.

**잘못된 예 (민감 정보와 검증되지 않은 문맥을 기록함):**

```typescript
console.log('Login attempt', { email, password });
logger.log(`User ${req.body.userId} requested ${req.originalUrl}`);
// 요청자가 다른 사용자 ID를 주장할 수 있으며 URL에 비밀 값이 있을 수 있다.
```

**올바른 예 (NestJS 11 JSON 로거와 CLS 문맥을 연결함):**

JSON 출력을 직접 구현하는 대신 내장 `ConsoleLogger`를 설정한다. Nest 로거의 두 번째 인자는 Pino의 데이터 인자와 같지 않다. 구조화된 필드는 첫 번째 메시지 객체에 넣는다.

```typescript
import { ConsoleLogger, Logger } from '@nestjs/common';
import { APP_INTERCEPTOR } from '@nestjs/core';
import { ClsModule, ClsService } from 'nestjs-cls';
import { randomUUID } from 'node:crypto';

@Injectable()
export class UserContextInterceptor implements NestInterceptor {
  constructor(private readonly cls: ClsService) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<unknown> {
    const req = context.switchToHttp().getRequest();
    this.cls.set('userId', req.user?.id);
    return next.handle();
  }
}

// 원본 입력과 임의 객체 대신 정해진 운영 필드만 받는다.
interface LogFields {
  resourceId?: string;
  durationMs?: number;
  errorCode?: string;
}

@Injectable()
export class ContextLogger {
  private readonly logger = new Logger(ContextLogger.name);
  constructor(private readonly cls: ClsService) {}

  log(event: string, fields: LogFields = {}): void {
    this.logger.log(this.record(event, fields));
  }

  error(event: string, fields: LogFields = {}): void {
    this.logger.error(this.record(event, fields));
  }

  private record(event: string, fields: LogFields) {
    return {
      event,
      requestId: this.cls.get('requestId'),
      userId: this.cls.get('userId'),
      resourceId: fields.resourceId,
      durationMs: fields.durationMs,
      errorCode: fields.errorCode,
    };
  }
}

@Module({
  imports: [
    ClsModule.forRoot({
      global: true,
      middleware: {
        mount: true,
        generateId: true,
        // 외부 요청자가 지정한 ID를 감사 로그의 식별자로 그대로 신뢰하지 않는다.
        idGenerator: () => randomUUID(),
        setup: (cls, _req, res) => {
          cls.set('requestId', cls.getId());
          res.setHeader('x-request-id', cls.getId());
        },
      },
    }),
  ],
  providers: [
    ContextLogger,
    { provide: APP_INTERCEPTOR, useClass: UserContextInterceptor },
  ],
  exports: [ContextLogger],
})
export class ObservabilityModule {}

// AppModule과 ContextLogger를 주입하는 기능 모듈에서 ObservabilityModule을 가져온다.
async function bootstrap() {
  const app = await NestFactory.create(AppModule, {
    logger: new ConsoleLogger({
      json: true,
      logLevels: process.env.NODE_ENV === 'production'
        ? ['log', 'warn', 'error', 'fatal']
        : ['log', 'warn', 'error', 'fatal', 'debug', 'verbose'],
    }),
  });
  await app.listen(3000);
}

@Injectable()
export class UsersService {
  constructor(private readonly logger: ContextLogger) {}

  recordCreated(userId: string): void {
    this.logger.log('user.created', { resourceId: userId });
  }
}
```

CLS 미들웨어는 가드보다 먼저 실행되므로 요청 ID만 초기화한다. 전역 `UserContextInterceptor`는 [인증 가드](security-use-guards.md)가 현재 사용자를 설정한 뒤 실행하며 사용자 문맥을 읽는 다른 인터셉터보다 먼저 등록한다. 공개 요청의 `userId`는 비어 있다. 인가 거부 요청은 인터셉터에 도달하지 않으므로 해당 감사 로그도 필요하면 인증 성공 직후 가드에서 CLS 사용자 ID를 설정하고 인가 가드를 그 뒤에 실행한다. 큐 작업·스케줄러는 HTTP CLS 문맥을 자동으로 상속하지 않으므로 작업 ID를 별도로 기록한다.

Pino를 이미 사용한다면 `nestjs-pino`의 `LoggerModule`을 등록하고 `bufferLogs: true`로 생성한 앱에 `app.useLogger(app.get(Logger))`를 적용한다. 이때 `Logger`는 `nestjs-pino`에서 가져온다. `PinoLogger`는 `logger.info({ resourceId }, 'user.created')`처럼 데이터가 첫 인자다. 요청·응답 직렬화는 필드를 제한하고, redaction 경로는 실제 출력 객체와 맞는지 테스트한다. 전체 body를 기록한 뒤 일부 키만 지우는 방식은 새 민감 필드를 놓치기 쉽다.

오류 원문·스택에는 SQL, URL, 고객 데이터가 포함될 수 있다. 필요한 상세 오류는 마스킹을 거친 제한된 운영 로그에만 남긴다. 필드 제거, 로그 보존 기간과 접근 권한은 서비스의 데이터 정책에 맞춘다.

참고: [NestJS 11 Logger](https://docs.nestjs.com/v11/techniques/logger), [NestJS CLS](https://papooch.github.io/nestjs-cls/setting-up-cls-context), [nestjs-pino](https://github.com/iamolegga/nestjs-pino)
