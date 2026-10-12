---
title: 구조화된 로깅을 사용한다
impact: MEDIUM-HIGH
impactDescription: "구조화된 로그로 효과적인 디버깅과 모니터링을 지원함"
tags: devops, logging, structured-logs, pino
---

## 구조화된 로깅을 사용한다

운영 환경에서는 NestJS Logger와 구조화된 JSON 출력을 사용한다. 요청 ID, 사용자 ID, 작업 같은 문맥 정보를 포함해 여러 서비스에 걸친 요청을 추적한다. console.log 사용을 피하고 적절한 로그 수준을 적용한다.

**잘못된 예 (운영 환경에서 console.log를 사용함):**

```typescript
// 운영 환경에서 console.log 사용
@Injectable()
export class UsersService {
  async createUser(dto: CreateUserDto): Promise<User> {
    console.log('Creating user:', dto);
    // 구조와 수준이 없고 운영 로그에 묻힘

    try {
      const user = await this.repo.save(dto);
      console.log('User created:', user.id);
      return user;
    } catch (error) {
      console.log('Error:', error); // 오류에 log 사용
      throw error;
    }
  }
}

// 민감한 데이터 로깅
console.log('Login attempt:', { email, password }); // 보안 위험!

// 일관되지 않은 로그 형식
logger.log('User ' + userId + ' created at ' + new Date());
// 구조가 없어 파싱하기 어려움
```

**올바른 예 (문맥 정보를 포함한 구조화된 로깅을 사용함):**

```typescript
// main.ts에서 로거 설정
async function bootstrap() {
  const app = await NestFactory.create(AppModule, {
    logger:
      process.env.NODE_ENV === 'production'
        ? ['error', 'warn', 'log']
        : ['error', 'warn', 'log', 'debug', 'verbose'],
  });
}

// 문맥을 포함하는 NestJS Logger 사용
@Injectable()
export class UsersService {
  private readonly logger = new Logger(UsersService.name);

  async createUser(dto: CreateUserDto): Promise<User> {
    this.logger.log('Creating user', { email: dto.email });

    try {
      const user = await this.repo.save(dto);
      this.logger.log('User created', { userId: user.id });
      return user;
    } catch (error) {
      this.logger.error('Failed to create user', error.stack, {
        email: dto.email,
      });
      throw error;
    }
  }
}

// JSON 출력을 위한 사용자 정의 로거
@Injectable()
export class JsonLogger implements LoggerService {
  log(message: string, context?: object): void {
    console.log(
      JSON.stringify({
        level: 'info',
        timestamp: new Date().toISOString(),
        message,
        ...context,
      }),
    );
  }

  error(message: string, trace?: string, context?: object): void {
    console.error(
      JSON.stringify({
        level: 'error',
        timestamp: new Date().toISOString(),
        message,
        trace,
        ...context,
      }),
    );
  }

  warn(message: string, context?: object): void {
    console.warn(
      JSON.stringify({
        level: 'warn',
        timestamp: new Date().toISOString(),
        message,
        ...context,
      }),
    );
  }

  debug(message: string, context?: object): void {
    console.debug(
      JSON.stringify({
        level: 'debug',
        timestamp: new Date().toISOString(),
        message,
        ...context,
      }),
    );
  }
}

// ClsModule로 요청 문맥 로깅
import { ClsModule, ClsService } from 'nestjs-cls';

@Module({
  imports: [
    ClsModule.forRoot({
      global: true,
      middleware: {
        mount: true,
        generateId: true,
      },
    }),
  ],
})
export class AppModule {}

// 요청 문맥을 설정하는 미들웨어
@Injectable()
export class RequestContextMiddleware implements NestMiddleware {
  constructor(private cls: ClsService) {}

  use(req: Request, res: Response, next: NextFunction): void {
    const requestId = req.headers['x-request-id'] || randomUUID();
    this.cls.set('requestId', requestId);
    this.cls.set('userId', req.user?.id);

    res.setHeader('x-request-id', requestId);
    next();
  }
}

// 요청 문맥을 포함하는 로거
@Injectable()
export class ContextLogger {
  constructor(private cls: ClsService) {}

  log(message: string, data?: object): void {
    console.log(
      JSON.stringify({
        level: 'info',
        timestamp: new Date().toISOString(),
        requestId: this.cls.get('requestId'),
        userId: this.cls.get('userId'),
        message,
        ...data,
      }),
    );
  }

  error(message: string, error: Error, data?: object): void {
    console.error(
      JSON.stringify({
        level: 'error',
        timestamp: new Date().toISOString(),
        requestId: this.cls.get('requestId'),
        userId: this.cls.get('userId'),
        message,
        error: error.message,
        stack: error.stack,
        ...data,
      }),
    );
  }
}

// 고성능 로깅을 위한 Pino 통합
import { LoggerModule } from 'nestjs-pino';

@Module({
  imports: [
    LoggerModule.forRoot({
      pinoHttp: {
        level: process.env.NODE_ENV === 'production' ? 'info' : 'debug',
        transport:
          process.env.NODE_ENV !== 'production'
            ? { target: 'pino-pretty' }
            : undefined,
        redact: ['req.headers.authorization', 'req.body.password'],
        serializers: {
          req: (req) => ({
            method: req.method,
            url: req.url,
            query: req.query,
          }),
          res: (res) => ({
            statusCode: res.statusCode,
          }),
        },
      },
    }),
  ],
})
export class AppModule {}

// Pino 사용 예
@Injectable()
export class UsersService {
  constructor(private logger: PinoLogger) {
    this.logger.setContext(UsersService.name);
  }

  async findOne(id: string): Promise<User> {
    this.logger.info({ userId: id }, 'Finding user');
    // Pino는 첫 인자로 데이터, 두 번째 인자로 메시지를 받음
  }
}
```

참고: [NestJS Logger](https://docs.nestjs.com/techniques/logger)
