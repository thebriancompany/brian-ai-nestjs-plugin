---
title: 공통 관심사에 인터셉터를 사용한다
impact: MEDIUM-HIGH
impactDescription: "공통 로직을 명확하게 분리함"
tags: api, interceptors, logging, caching
---

## 공통 관심사에 인터셉터를 사용한다

인터셉터는 비즈니스 로직에 코드를 섞지 않고 응답 변환, 로깅, 캐싱, 성능 측정을 처리할 수 있다. 라우트 핸들러의 실행을 감싸므로 요청과 응답 스트림 모두에 접근할 수 있다.

**잘못된 예 (모든 메서드에서 로깅과 응답 변환을 수행함):**

```typescript
// 모든 컨트롤러 메서드에서 로깅함
@Controller('users')
export class UsersController {
  @Get()
  async findAll(): Promise<User[]> {
    const start = Date.now();
    this.logger.log('findAll called');

    const users = await this.usersService.findAll();

    this.logger.log(`findAll completed in ${Date.now() - start}ms`);
    return users;
  }

  @Get(':id')
  async findOne(@Param('id') id: string): Promise<User> {
    const start = Date.now();
    this.logger.log(`findOne called with id: ${id}`);

    const user = await this.usersService.findOne(id);

    this.logger.log(`findOne completed in ${Date.now() - start}ms`);
    return user;
  }
  // 모든 메서드에서 반복됨!
}

// 응답을 수동으로 감쌈
@Get()
async findAll(): Promise<{ data: User[]; meta: Meta }> {
  const users = await this.usersService.findAll();
  return {
    data: users,
    meta: { timestamp: new Date(), count: users.length },
  };
}
```

**올바른 예 (인터셉터로 공통 관심사를 처리함):**

```typescript
// 로깅 인터셉터
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  private readonly logger = new Logger('HTTP');

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const { method, url, body } = request;
    const now = Date.now();

    return next.handle().pipe(
      tap({
        next: (data) => {
          const response = context.switchToHttp().getResponse();
          this.logger.log(
            `${method} ${url} ${response.statusCode} - ${Date.now() - now}ms`,
          );
        },
        error: (error) => {
          this.logger.error(
            `${method} ${url} ${error.status || 500} - ${Date.now() - now}ms`,
            error.stack,
          );
        },
      }),
    );
  }
}

// 응답 변환 인터셉터
@Injectable()
export class TransformInterceptor<T> implements NestInterceptor<T, Response<T>> {
  intercept(context: ExecutionContext, next: CallHandler): Observable<Response<T>> {
    return next.handle().pipe(
      map((data) => ({
        data,
        meta: {
          timestamp: new Date().toISOString(),
          path: context.switchToHttp().getRequest().url,
        },
      })),
    );
  }
}

// 시간 초과 인터셉터
@Injectable()
export class TimeoutInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      timeout(5000),
      catchError((err) => {
        if (err instanceof TimeoutError) {
          throw new RequestTimeoutException('Request timed out');
        }
        throw err;
      }),
    );
  }
}

// 전역 또는 컨트롤러별 적용
@Module({
  providers: [
    { provide: APP_INTERCEPTOR, useClass: LoggingInterceptor },
    { provide: APP_INTERCEPTOR, useClass: TransformInterceptor },
  ],
})
export class AppModule {}

// 또는 컨트롤러별 적용
@Controller('users')
@UseInterceptors(LoggingInterceptor)
export class UsersController {
  @Get()
  async findAll(): Promise<User[]> {
    // 비즈니스 로직만 간결하게 유지
    return this.usersService.findAll();
  }
}

// 사용자나 요청 문맥에 따라 내용이 달라지지 않는 공개 GET 응답에만 사용한다.
// CacheModule을 이 프로바이더의 모듈에 import해야 한다.
import { CACHE_MANAGER } from '@nestjs/cache-manager';
import type { Cache } from 'cache-manager';

@Injectable()
export class HttpCacheInterceptor implements NestInterceptor {
  private readonly logger = new Logger(HttpCacheInterceptor.name);

  constructor(
    @Inject(CACHE_MANAGER) private cacheManager: Cache,
    private reflector: Reflector,
  ) {}

  async intercept(context: ExecutionContext, next: CallHandler): Promise<Observable<unknown>> {
    const request = context.switchToHttp().getRequest();
    const enabled = this.reflector.getAllAndOverride<boolean>('publicCache', [
      context.getHandler(), context.getClass(),
    ]);
    // 인증 사용자, 쿠키가 있는 요청, 비공개 경로는 캐시를 공유하지 않는다.
    if (!enabled || request.method !== 'GET' || request.user ||
        request.headers.authorization || request.headers.cookie) {
      return next.handle();
    }

    const cacheKey = this.generateKey(request);
    const ttl = this.reflector.get<number>('cacheTTL', context.getHandler()) ?? 60_000;
    try {
      const cached = await this.cacheManager.get(cacheKey);
      if (cached !== undefined && cached !== null) return of(cached);
    } catch {
      this.logger.warn('Cache read failed');
    }

    return next.handle().pipe(
      mergeMap(async response => {
        try {
          await this.cacheManager.set(cacheKey, response, ttl);
        } catch {
          this.logger.warn('Cache write failed');
        }
        return response;
      }),
    );
  }

  private generateKey(request: Request): string {
    return JSON.stringify([
      'public', request.headers.host, request.originalUrl ?? request.url,
      request.headers.accept, request.headers['accept-language'],
    ]);
  }
}

// 개인화·테넌트별 응답에는 이 공개 캐시를 적용하지 않는다.
// TTL 단위는 현재 cache-manager에서 밀리초다.
@Get()
@SetMetadata('publicCache', true)
@SetMetadata('cacheTTL', 60_000)
@UseInterceptors(HttpCacheInterceptor)
async findAll(): Promise<PublicCategoryDto[]> {
  return this.categoriesService.findPublicCategories();
}

// 오류 매핑 인터셉터
@Injectable()
export class ErrorMappingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      catchError((error) => {
        if (error instanceof EntityNotFoundError) {
          throw new NotFoundException(error.message);
        }
        if (error instanceof QueryFailedError) {
          if (error.message.includes('duplicate')) {
            throw new ConflictException('Resource already exists');
          }
        }
        throw error;
      }),
    );
  }
}
```

참고: [NestJS Interceptors](https://docs.nestjs.com/interceptors)
