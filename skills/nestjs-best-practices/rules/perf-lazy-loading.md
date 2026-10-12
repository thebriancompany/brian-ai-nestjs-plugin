---
title: 큰 모듈에 지연 로딩을 사용한다
impact: MEDIUM
impactDescription: "대규모 애플리케이션의 시작 시간을 개선함"
tags: performance, lazy-loading, modules, optimization
---

## 큰 모듈에 지연 로딩을 사용한다

NestJS는 처음 사용할 때까지 초기화를 미루는 모듈 지연 로딩을 지원한다. 일부 기능을 드물게 사용하는 대규모 애플리케이션, 콜드 스타트 시간이 중요한 서버리스 배포, 특정 모듈의 초기화 비용이 큰 상황에 유용하다.

**잘못된 예 (모든 모듈을 시작할 때 로딩함):**

```typescript
// 대규모 앱에서 모든 모듈을 시작 시 로딩
@Module({
  imports: [
    UsersModule,
    OrdersModule,
    PaymentsModule,
    ReportsModule, // 무겁고 드물게 사용됨
    AnalyticsModule, // 무겁고 드물게 사용됨
    AdminModule, // 관리자만 사용함
    LegacyModule, // 드물게 사용하는 마이그레이션 모듈
    BulkImportModule, // 한 달에 한 번 사용
  ],
})
export class AppModule {}

// 사용하지 않아도 시작 시 모든 모듈을 초기화함
// 서버리스에서 콜드 스타트가 느림
// 사용하지 않는 모듈로 메모리 낭비
```

**올바른 예 (드물게 사용하는 모듈을 지연 로딩함):**

```typescript
// 선택적 모듈에는 LazyModuleLoader 사용
import { LazyModuleLoader } from '@nestjs/core';

@Injectable()
export class ReportsService {
  constructor(private lazyModuleLoader: LazyModuleLoader) {}

  async generateReport(type: string): Promise<Report> {
    // 필요할 때만 모듈 로딩
    const { ReportsModule } = await import('./reports/reports.module');
    const moduleRef = await this.lazyModuleLoader.load(() => ReportsModule);

    const reportsService = moduleRef.get(ReportsGeneratorService);
    return reportsService.generate(type);
  }
}

// 캐싱과 함께 관리 기능을 지연 로딩
@Injectable()
export class AdminService {
  private adminModule: ModuleRef | null = null;

  constructor(private lazyModuleLoader: LazyModuleLoader) {}

  private async getAdminModule(): Promise<ModuleRef> {
    if (!this.adminModule) {
      const { AdminModule } = await import('./admin/admin.module');
      this.adminModule = await this.lazyModuleLoader.load(() => AdminModule);
    }
    return this.adminModule;
  }

  async runAdminTask(task: string): Promise<void> {
    const moduleRef = await this.getAdminModule();
    const taskRunner = moduleRef.get(AdminTaskRunner);
    await taskRunner.run(task);
  }
}

// 재사용 가능한 지연 로딩 서비스
@Injectable()
export class ModuleLoaderService {
  private loadedModules = new Map<string, ModuleRef>();

  constructor(private lazyModuleLoader: LazyModuleLoader) {}

  async load<T>(
    key: string,
    importFn: () => Promise<{ default: Type<T> } | Type<T>>,
  ): Promise<ModuleRef> {
    if (!this.loadedModules.has(key)) {
      const module = await importFn();
      const moduleType = 'default' in module ? module.default : module;
      const moduleRef = await this.lazyModuleLoader.load(() => moduleType);
      this.loadedModules.set(key, moduleRef);
    }
    return this.loadedModules.get(key)!;
  }
}

// 시작 후 백그라운드에서 모듈 미리 로딩
@Injectable()
export class ModulePreloader implements OnApplicationBootstrap {
  constructor(private lazyModuleLoader: LazyModuleLoader) {}

  async onApplicationBootstrap(): Promise<void> {
    setTimeout(async () => {
      await this.preloadModule(() => import('./reports/reports.module'));
    }, 5000); // 시작 후 5초
  }

  private async preloadModule(importFn: () => Promise<any>): Promise<void> {
    try {
      const module = await importFn();
      const moduleType = module.default || Object.values(module)[0];
      await this.lazyModuleLoader.load(() => moduleType);
    } catch (error) {
      console.warn('Failed to preload module', error);
    }
  }
}
```

참고: [NestJS Lazy Loading Modules](https://docs.nestjs.com/fundamentals/lazy-loading-modules)
