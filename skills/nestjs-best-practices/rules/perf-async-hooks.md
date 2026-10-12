---
title: 비동기 생명주기 훅을 올바르게 사용한다
impact: HIGH
impactDescription: "잘못된 비동기 처리는 애플리케이션 시작을 막음"
tags: performance, lifecycle, async, hooks
---

## 비동기 생명주기 훅을 올바르게 사용한다

NestJS의 생명주기 훅(`onModuleInit`, `onApplicationBootstrap` 등)은 비동기 작업을 지원한다. 하지만 잘못 사용하면 애플리케이션 시작을 막거나 경쟁 상태를 일으킬 수 있다. 생명주기의 순서를 이해하고 훅을 적절하게 사용한다.

**잘못된 예 (await 없이 비동기 작업을 실행하고 기다리지 않음):**

```typescript
// await 없이 비동기 작업을 실행하고 기다리지 않음
@Injectable()
export class DatabaseService implements OnModuleInit {
  onModuleInit() {
    // 작업을 기다리지 않아 DB가 준비되기 전에 앱이 시작됨!
    this.connect();
  }

  private async connect() {
    await this.pool.connect();
    console.log('Database connected');
  }
}

// 생성자에서 오래 걸리는 블로킹 작업 수행
@Injectable()
export class ConfigService {
  private config: Config;

  constructor() {
    // 전체 모듈 인스턴스 생성을 동기적으로 막음
    this.config = fs.readFileSync('config.json');
  }
}
```

**올바른 예 (비동기 훅에서 Promise를 반환함):**

```typescript
// 비동기 훅에서 Promise 반환
@Injectable()
export class DatabaseService implements OnModuleInit {
  private pool: Pool;

  async onModuleInit(): Promise<void> {
    // NestJS는 완료될 때까지 기다린 후 계속 진행함
    await this.pool.connect();
    console.log('Database connected');
  }

  async onModuleDestroy(): Promise<void> {
    // 종료 시 자원 정리
    await this.pool.end();
    console.log('Database disconnected');
  }
}

// 모듈 간 의존성이 있으면 onApplicationBootstrap 사용
@Injectable()
export class CacheWarmerService implements OnApplicationBootstrap {
  constructor(
    private cache: CacheService,
    private products: ProductsService,
  ) {}

  async onApplicationBootstrap(): Promise<void> {
    // 모든 모듈을 초기화했으므로 안전하게 캐시를 준비할 수 있음
    const products = await this.products.findPopular();
    await this.cache.warmup(products);
  }
}

// 무거운 초기화는 생성자 대신 비동기 훅에서 수행
@Injectable()
export class ConfigService implements OnModuleInit {
  private config: Config;

  constructor() {
    // 생성자는 동기적이고 빠르게 유지
  }

  async onModuleInit(): Promise<void> {
    // 생명주기 훅에서 비동기 로딩
    this.config = await this.loadConfig();
  }

  private async loadConfig(): Promise<Config> {
    const file = await fs.promises.readFile('config.json');
    return JSON.parse(file.toString());
  }

  get<T>(key: string): T {
    return this.config[key];
  }
}

// main.ts에서 종료 훅 활성화
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.enableShutdownHooks(); // SIGTERM/SIGINT 처리 활성화
  await app.listen(3000);
}
```

참고: [NestJS Lifecycle Events](https://docs.nestjs.com/fundamentals/lifecycle-events)
