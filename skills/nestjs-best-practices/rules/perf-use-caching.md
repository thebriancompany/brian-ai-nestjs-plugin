---
title: 캐싱을 전략적으로 사용한다
impact: HIGH
impactDescription: "데이터베이스 부하와 응답 시간을 크게 줄임"
tags: performance, caching, redis, optimization
---

## 캐싱을 전략적으로 사용한다

비용이 큰 작업, 자주 접근하는 데이터, 외부 API 호출에 캐싱을 적용한다. 적절한 TTL과 캐시 무효화 전략을 갖춘 NestJS CacheModule을 사용한다. 모든 데이터를 캐싱하지 말고 효과가 큰 부분에 집중한다.

**잘못된 예 (캐싱을 전혀 하지 않거나 모든 것을 캐싱함):**

```typescript
// 비용이 크고 반복되는 쿼리에 캐싱이 없음
@Injectable()
export class ProductsService {
  async getPopular(): Promise<Product[]> {
    // 매 요청마다 복잡한 집계 쿼리 실행
    return this.productsRepo
      .createQueryBuilder('p')
      .leftJoin('p.orders', 'o')
      .select('p.*, COUNT(o.id) as orderCount')
      .groupBy('p.id')
      .orderBy('orderCount', 'DESC')
      .limit(20)
      .getMany();
  }
}

// 고려 없이 모든 데이터를 캐싱
@Injectable()
export class UsersService {
  @CacheKey('users')
  @CacheTTL(3600)
  @UseInterceptors(CacheInterceptor)
  async findAll(): Promise<User[]> {
    // 데이터가 자주 바뀌면 사용자 목록을 1시간 캐싱하는 것은 잘못됨
    return this.usersRepo.find();
  }
}
```

**올바른 예 (적절한 무효화 전략과 함께 캐싱함):**

```typescript
// 캐싱 모듈 설정
@Module({
  imports: [
    CacheModule.registerAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        stores: [
          new KeyvRedis(config.get('REDIS_URL')),
        ],
        ttl: 60 * 1000, // 기본 60초
      }),
    }),
  ],
})
export class AppModule {}

// 세밀한 제어를 위한 수동 캐싱
@Injectable()
export class ProductsService {
  constructor(
    @Inject(CACHE_MANAGER) private cache: Cache,
    private productsRepo: ProductRepository,
  ) {}

  async getPopular(): Promise<Product[]> {
    const cacheKey = 'products:popular';

    // 먼저 캐시 조회
    const cached = await this.cache.get<Product[]>(cacheKey);
    if (cached) return cached;

    // 캐시에 없으면 조회 후 저장
    const products = await this.fetchPopularProducts();
    await this.cache.set(cacheKey, products, 5 * 60 * 1000); // TTL 5분
    return products;
  }

  // 변경 시 캐시 무효화
  async updateProduct(id: string, dto: UpdateProductDto): Promise<Product> {
    const product = await this.productsRepo.save({ id, ...dto });
    await this.cache.del('products:popular'); // 무효화
    return product;
  }
}

// 공개 카테고리에만 적용한다. 인증·테넌트별 응답에는 별도 키/무효화 정책이 필요하다.
// 자동 인터셉터를 이용한 데코레이터 기반 캐싱
@Controller('categories')
@UseInterceptors(CacheInterceptor)
export class CategoriesController {
  @Get()
  @CacheTTL(30 * 60 * 1000) // 30분: 카테고리는 드물게 변경됨
  findAll(): Promise<Category[]> {
    return this.categoriesService.findAll();
  }

  // 경로의 :id가 기본 캐시 키에 포함되도록 고정 CacheKey를 지정하지 않는다.
  @Get(':id')
  @CacheTTL(60 * 1000) // 1분
  findOne(@Param('id') id: string): Promise<Category> {
    return this.categoriesService.findOne(id);
  }
}

// 이벤트 기반 캐시 무효화
@Injectable()
export class CacheInvalidationService {
  constructor(@Inject(CACHE_MANAGER) private cache: Cache) {}

  @OnEvent('product.created')
  @OnEvent('product.updated')
  @OnEvent('product.deleted')
  async invalidateProductCaches(event: ProductEvent) {
    await Promise.all([
      this.cache.del('products:popular'),
      this.cache.del(`product:${event.productId}`),
    ]);
  }
}
```

참고: [NestJS Caching](https://docs.nestjs.com/techniques/caching)
