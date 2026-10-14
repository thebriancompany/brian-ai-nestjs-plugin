---
title: 캐싱을 전략적으로 사용한다
impact: HIGH
impactDescription: "정합성 요구에 맞춰 반복 조회 비용을 줄임"
tags: performance, caching, redis, optimization
---

## 캐싱을 전략적으로 사용한다

캐싱은 데이터가 얼마나 오래되어도 되는지 먼저 정한 뒤 적용한다. 공개 목록처럼 잠시 오래된 값을 허용하는 조회와 권한·재고·결제 판단을 구분한다. 키에는 응답에 영향을 주는 테넌트, 사용자, 필터, 언어와 스키마 버전을 포함한다. 기본 HTTP URL 키는 사용자별 응답을 구분하지 않는다.

**잘못된 예 (사용자별 응답을 하나의 키로 공유함):**

```typescript
@Controller('users')
export class UsersController {
  @Get('me')
  @CacheKey('current-user') // 모든 사용자가 같은 캐시를 읽음
  @UseInterceptors(CacheInterceptor)
  me(@Req() req: AuthenticatedRequest) {
    return this.usersService.findProfile(req.user.id);
  }
}
// 일반 서비스 메서드에 @UseInterceptors를 붙여도 직접 호출을 가로채지 않는다.
```

**올바른 예 (공개 데이터의 TTL과 캐시 장애 정책을 명시함):**

아래는 `@nestjs/cache-manager` 3 / `cache-manager` 7, `keyv` 5.6, `@keyv/redis` 5.1.6 / `@redis/client` 5.12의 구성이다. TTL은 밀리초다. Nest 캐시 모듈 3은 Keyv 6 인스턴스를 저장소로 다시 감쌀 수 있으므로 이 예제는 호환되는 Keyv 5 조합을 사용한다. 연결·명령 시간 제한 옵션도 설치된 버전의 API를 확인하고 적용한다.

```typescript
import { Inject, Injectable, Logger, Module } from '@nestjs/common';
import { CACHE_MANAGER, CacheModule } from '@nestjs/cache-manager';
import type { Cache } from 'cache-manager';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { createKeyv } from '@keyv/redis';

@Module({
  imports: [CacheModule.registerAsync({
    imports: [ConfigModule],
    inject: [ConfigService],
    useFactory: (config: ConfigService) => {
      const logger = new Logger('RedisCache');
      const store = createKeyv({
        url: config.getOrThrow<string>('REDIS_URL'),
        socket: { connectTimeout: 500, socketTimeout: 500, reconnectStrategy: false },
        disableOfflineQueue: true,
        commandsQueueMaxLength: 1000,
        commandOptions: { timeout: 500 }, // 전송 대기 제한. 응답 대기는 별도로 제한한다.
      }, { connectionTimeout: 500, throwOnErrors: true });
      // 연결 오류 이벤트는 서비스 메서드의 try/catch 밖에서도 발생한다.
      store.on('error', () => logger.warn('Redis cache unavailable'));
      return { stores: [store], ttl: 30_000 };
    },
  })],
  exports: [CacheModule],
})
export class AppCacheModule {}
// ProductsModule에서 AppCacheModule을 imports 한다.
// ConfigModule.forRoot()와 환경 값 검증은 루트 구성에서 수행한다.

interface ProductSummary {
  id: string;
  name: string;
}

@Injectable()
export class ProductsService {
  private readonly logger = new Logger(ProductsService.name);
  private readonly popularKey = 'public:products:popular:v1';

  constructor(
    @Inject(CACHE_MANAGER) private readonly cache: Cache,
    private readonly productsRepo: ProductRepository,
  ) {}

  async getPopular(): Promise<ProductSummary[]> {
    try {
      const cached = await this.withinCacheBudget(
        this.cache.get<ProductSummary[]>(this.popularKey),
      );
      if (cached !== undefined && cached !== null) return cached;
    } catch {
      this.logger.warn('Popular products cache read failed');
    }

    // 공개 필드만 가진 JSON DTO를 반환하는 제한된 조회다.
    // DB 오류는 캐시 오류와 달리 호출자에게 전파한다.
    const products = await this.productsRepo.findPopularSummaries(20);
    try {
      await this.withinCacheBudget(this.cache.set(this.popularKey, products, 30_000));
    } catch {
      this.logger.warn('Popular products cache write failed');
    }
    return products;
  }

  // 변경 트랜잭션이 커밋된 뒤 생성/수정/삭제 및 순위 변경 경로에서 호출한다.
  async invalidatePopular(): Promise<void> {
    try {
      await this.withinCacheBudget(this.cache.del(this.popularKey));
    } catch {
      // 이미 커밋된 변경을 실패로 응답해 클라이언트 재시도를 유발하지 않는다.
      this.logger.warn('Popular products cache invalidation failed');
    }
  }

  private async withinCacheBudget<T>(work: Promise<T>): Promise<T> {
    let timer: ReturnType<typeof setTimeout> | undefined;
    try {
      return await Promise.race([
        work,
        new Promise<never>((_, reject) => {
          timer = setTimeout(() => reject(new Error('Cache deadline exceeded')), 600);
        }),
      ]);
    } finally {
      if (timer) clearTimeout(timer);
    }
  }
}
```

이 예제는 TTL 동안 오래된 목록을 허용한다. `del()` 직전에 시작한 조회가 이전 값을 다시 `set()`할 수 있으므로 삭제만으로 즉시 일관성이 보장되지는 않는다. 허용 시간을 계산할 때 조회 시간도 포함한다. 즉시 반영이 필요한 데이터는 원본에서 읽거나 데이터 버전/원자적 갱신을 포함한 별도 전략을 검증한다. 주문 생성이나 JWT 권한 판단에 이 공개 목록 캐시를 사용하지 않는다.

이 예제의 연결·소켓·요청별 캐시 대기 제한은 서비스 지연 예산에 맞춰 조정한다. `socketTimeout`은 개별 명령의 전체 deadline이 아니라 연결의 무통신 시간을 제한하며 유휴 연결도 닫는다. 다음 조회는 다시 연결한다. `withinCacheBudget`은 요청의 캐시 대기만 중단하고 이미 보낸 작업을 취소하지 않으므로 늦은 캐시 쓰기도 가능하다. 큐 크기 제한·소켓 종료와 함께 검증하고 반드시 실행을 취소해야 하는 업무 작업에 이 패턴을 사용하지 않는다. catch는 거부된 Promise를 처리할 뿐 이벤트 오류를 해결하지 않는다.

장애 시 DB로 우회하는 부하를 용량·요청 제한과 함께 검증한다. 같은 키의 동시 누락이 실제 병목이면 요청 합치기를 추가하며, 프로세스 내부 합치기만으로 여러 인스턴스의 중복 조회를 막을 수는 없다.

HTTP 자동 캐싱은 공개 GET 경로에 한정한다. `:id`나 쿼리별 응답에 고정 `@CacheKey()`를 지정하면 값이 섞인다. 사용자별 캐시가 필요하면 인증 이후 식별자를 포함한 키를 만들고 권한 변경 시 무효화한다. 엔티티 인스턴스나 `Date`가 저장소를 왕복한 뒤에도 같은 타입이라고 가정하지 않는다.

검증: 캐시 누락/빈 배열/적중, Redis 실패 시 DB 조회, 저장 실패 시 정상 응답, 커밋 후 무효화, 읽기와 쓰기의 경합, 사용자·테넌트 간 분리를 확인한다. 무효화 신뢰성이 필요한 다중 프로세스 환경에서는 로컬 이벤트만으로 전파되었다고 가정하지 않는다.

참고: [NestJS Caching](https://docs.nestjs.com/techniques/caching), [Keyv Redis Adapter](https://keyv.org/docs/storage-adapters/redis/)
