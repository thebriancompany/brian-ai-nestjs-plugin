---
title: 올바른 모듈 공유 패턴을 사용한다
impact: CRITICAL
impactDescription: "중복 인스턴스, 메모리 누수, 상태 불일치를 방지함"
tags: architecture, modules, sharing, exports
---

## 올바른 모듈 공유 패턴을 사용한다

NestJS 모듈은 기본적으로 싱글턴이다. 모듈에서 서비스를 올바르게 내보내고 다른 곳에서 해당 모듈을 가져오면 같은 인스턴스를 공유한다. 반면 여러 모듈에 같은 서비스를 프로바이더로 등록하면 각각 별도 인스턴스가 생성되어 메모리 낭비, 상태 불일치, 이해하기 어려운 동작이 발생한다. 항상 서비스를 전용 모듈에 캡슐화하고 명시적으로 내보낸 뒤, 필요한 곳에서 그 모듈을 가져온다.

**잘못된 예 (여러 모듈에서 같은 서비스를 프로바이더로 등록함):**

```typescript
// StorageService를 여러 모듈에 직접 등록하는 잘못된 예
// storage.service.ts
@Injectable()
export class StorageService {
  private cache = new Map(); // 각 인스턴스가 별도 상태를 가짐!

  store(key: string, value: any) {
    this.cache.set(key, value);
  }
}

// app.module.ts
@Module({
  providers: [StorageService], // 인스턴스 1
  controllers: [AppController],
})
export class AppModule {}

// videos.module.ts
@Module({
  providers: [StorageService], // 인스턴스 2: AppModule의 인스턴스와 다름!
  controllers: [VideosController],
})
export class VideosModule {}

// 문제점:
// 1. 별개의 StorageService 인스턴스가 두 개 존재함
// 2. VideosModule의 cache.set()은 AppModule의 캐시에 영향을 주지 않음
// 3. 중복 인스턴스로 메모리를 낭비함
// 4. 상태가 동기화되지 않아 디버깅이 매우 어려움
```

**올바른 예 (전용 모듈에서 서비스를 내보냄):**

```typescript
// storage/storage.module.ts
@Module({
  providers: [StorageService],
  exports: [StorageService], // 가져오는 모듈에서 사용할 수 있도록 내보냄
})
export class StorageModule {}

// videos/videos.module.ts
@Module({
  imports: [StorageModule], // 서비스가 아니라 모듈을 가져옴
  controllers: [VideosController],
  providers: [VideosService],
})
export class VideosModule {}

// channels/channels.module.ts
@Module({
  imports: [StorageModule], // 같은 인스턴스를 공유함
  controllers: [ChannelsController],
  providers: [ChannelsService],
})
export class ChannelsModule {}

// app.module.ts
@Module({
  imports: [
    StorageModule, // AppModule 자체에 StorageService가 필요한 경우에만 가져옴
    VideosModule,
    ChannelsModule,
  ],
})
export class AppModule {}

// 이제 모든 모듈이 같은 StorageService 인스턴스를 공유함
```

**@Global()을 사용할 때 (제한적으로 사용):**

```typescript
// 애플리케이션 전반의 공통 관심사에만 사용
@Global()
@Module({
  providers: [ConfigService, LoggerService],
  exports: [ConfigService, LoggerService],
})
export class CoreModule {}

// AppModule에서 한 번만 가져옴
@Module({
  imports: [CoreModule], // 전역으로 등록되어 어디서나 사용 가능
})
export class AppModule {}

// 다른 모듈은 CoreModule을 가져올 필요가 없음
@Module({
  controllers: [UsersController],
  providers: [UsersService], // 모듈을 가져오지 않아도 ConfigService를 주입할 수 있음
})
export class UsersModule {}

// 주의: 모든 것을 전역으로 만들지 말 것!
// - 의존성이 숨겨짐 (imports만으로 모듈에 필요한 것을 알 수 없음)
// - 테스트가 어려워짐
// - 설정, 로깅, 데이터베이스 연결 등에 한정해서 사용
```

**모듈 다시 내보내기 패턴:**

```typescript
// common.module.ts: 공유 유틸리티
@Module({
  providers: [DateService, ValidationService],
  exports: [DateService, ValidationService],
})
export class CommonModule {}

// core.module.ts: 편의를 위해 공통 모듈을 다시 내보냄
@Module({
  imports: [CommonModule, DatabaseModule],
  exports: [CommonModule, DatabaseModule], // 사용하는 쪽을 위해 다시 내보냄
})
export class CoreModule {}

// feature.module.ts: CoreModule을 가져오면 두 모듈 모두 사용 가능
@Module({
  imports: [CoreModule], // CommonModule과 DatabaseModule을 모두 사용 가능
  controllers: [FeatureController],
})
export class FeatureModule {}
```

참고: [NestJS Modules](https://docs.nestjs.com/modules#shared-modules)
