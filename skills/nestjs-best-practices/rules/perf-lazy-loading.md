---
title: 선택적 프로바이더에 지연 로딩을 사용한다
impact: MEDIUM
impactDescription: "측정된 초기화 비용을 첫 사용 시점으로 미룸"
tags: performance, lazy-loading, modules, optimization
---

## 선택적 프로바이더에 지연 로딩을 사용한다

시작 시간이 실제 병목이고 기능을 드물게 사용한다면 `LazyModuleLoader`로 해당 프로바이더의 생성을 미룰 수 있다. 첫 요청 지연이 늘어나는 비용도 측정한다. 일반적인 기능 모듈을 시작 시 가져오는 것은 정상적인 기본값이며, 크다는 이유만으로 모두 지연 로딩하지 않는다.

컨트롤러·리졸버·게이트웨이·미들웨어는 이 방식으로 나중에 등록할 수 없다. 경로와 가드는 시작 시 등록하고, 그 경로에서 호출하는 계산 서비스를 지연 로딩한다. 지연 모듈의 생명주기 훅도 호출되지 않으므로 DB 연결이나 종료 훅에 의존하는 모듈에 그대로 적용하지 않는다.

**잘못된 예 (필수 경로 등록까지 지연하고 훅에 초기화를 맡김):**

```typescript
@Module({
  controllers: [ReportsController], // 늦게 로딩해도 경로가 등록되지 않음
  providers: [ReportsGeneratorService],
})
export class ReportsModule {}

@Injectable()
export class ReportsGeneratorService implements OnModuleInit {
  async onModuleInit(): Promise<void> {
    await this.loadTemplates(); // 지연 모듈에서는 호출되지 않음
  }
}
```

**올바른 예 (기존 경로에서 선택적 계산 서비스를 지연 로딩함):**

```typescript
import { Injectable } from '@nestjs/common';
import { LazyModuleLoader } from '@nestjs/core';
import type { ReportsGeneratorService } from './reports/reports-generator.service';

// ReportsFacade는 시작 시 등록된 모듈의 프로바이더다.
// ReportsModule은 시작 모듈의 imports 목록에 넣지 않는다.
@Injectable()
export class ReportsFacade {
  private loading?: Promise<ReportsGeneratorService>;

  constructor(private readonly lazyModuleLoader: LazyModuleLoader) {}

  async generateReport(type: ReportType): Promise<Report> {
    const generator = await this.getGenerator();
    return generator.generate(type);
  }

  private getGenerator(): Promise<ReportsGeneratorService> {
    // 동시에 들어온 최초 요청도 하나의 로딩 Promise를 공유한다.
    // 실패한 Promise도 유지한다. Nest 내부의 부분 초기화 상태를 재사용하지 않는다.
    return this.loading ??= this.loadGenerator();
  }

  private async loadGenerator(): Promise<ReportsGeneratorService> {
    const { ReportsModule } = await import('./reports/reports.module');
    const { ReportsGeneratorService } = await import('./reports/reports-generator.service');
    const moduleRef = await this.lazyModuleLoader.load(() => ReportsModule);
    return moduleRef.get(ReportsGeneratorService, { strict: true });
  }
}

// reports/reports.module.ts (별도 파일)
import { Module } from '@nestjs/common';
import { ReportsGeneratorService } from './reports-generator.service';

@Module({
  providers: [ReportsGeneratorService],
  exports: [ReportsGeneratorService],
})
export class ReportsModule {}
// ReportsGeneratorService는 훅 초기화나 장기 연결이 필요 없는 싱글턴 계산 서비스다.
```

Nest는 로딩한 모듈을 캐싱한다. 프로바이더 초기화 실패 후에도 내부 모듈 상태가 남을 수 있어, 이 예제는 실패를 유지하고 해당 기능의 복구에 앱 재시작을 요구한다. 필드를 `undefined`로 되돌려도 Nest 내부 상태는 초기화되지 않으므로 단순 재시도를 안전한 복구로 간주하지 않는다. 실패를 관찰·알림하고 재시작 정책을 정한다. 별도의 범용 모듈 레지스트리나 타이머 기반 프리로더는 필요가 확인될 때만 추가한다. 서비스 클래스도 동적 import하고 타입은 `import type`으로 가져와, 정적 import가 무거운 의존성을 먼저 로딩하지 않게 한다. 번들러의 실제 청크 분할 여부는 빌드 결과로 확인한다. 요청/일시적 스코프가 필요하면 `ModuleRef.get()` 대신 해당 요청 컨텍스트를 전달한 `resolve()` 전략을 별도로 설계한다.

검증: 최초 동시 요청의 모듈 초기화 횟수와 실제 라우트 등록, 첫 요청과 후속 요청 지연을 확인한다. 실제 Nest 프로바이더 생성 실패를 주입해 후속 요청도 같은 실패를 받고 부분 초기화 인스턴스를 사용하지 않는지, 원인을 제거하고 앱을 재시작한 뒤 복구되는지 확인한다.

참고: [NestJS Lazy Loading Modules](https://docs.nestjs.com/fundamentals/lazy-loading-modules)
