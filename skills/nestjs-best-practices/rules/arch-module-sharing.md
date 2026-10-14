---
title: 올바른 모듈 공유 패턴을 사용한다
impact: CRITICAL
impactDescription: "의도하지 않은 중복 인스턴스와 상태 불일치를 방지함"
tags: architecture, modules, sharing, exports
---

## 올바른 모듈 공유 패턴을 사용한다

같은 애플리케이션 컨텍스트에서 같은 모듈이 내보내는 기본 스코프 프로바이더는 그 모듈을 가져오는 소비자들이 공유한다. 여러 모듈의 `providers`에 같은 클래스를 각각 등록하면 별도 인스턴스가 생긴다. 공유가 목적이라면 소유 모듈에서 내보내고 필요한 모듈에서 가져온다. 별도 프로세스, 별도 애플리케이션 컨텍스트, 요청/트랜지언트 스코프, 서로 다른 동적 모듈 등록까지 하나의 인스턴스를 보장하는 것은 아니다.

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
// 애플리케이션 전반의 자체 로깅 프로바이더에만 사용
@Global()
@Module({
  providers: [AppLogger],
  exports: [AppLogger],
})
export class CoreModule {}

// 다른 모듈은 CoreModule을 가져올 필요가 없음
@Module({
  controllers: [UsersController],
  providers: [UsersService], // CoreModule을 가져오지 않아도 AppLogger를 주입할 수 있음
})
export class UsersModule {}

// AppModule에서 한 번만 가져옴
@Module({
  imports: [CoreModule, UsersModule], // 전역 등록 + 기능 모듈 등록
})
export class AppModule {}

// 주의: 모든 것을 전역으로 만들지 말 것!
// - 의존성이 숨겨짐 (imports만으로 모듈에 필요한 것을 알 수 없음)
// - 테스트가 어려워짐
// - 설정, 로깅, 데이터베이스 연결 등에 한정해서 사용
```

`@nestjs/config`의 `ConfigService`를 직접 다시 등록하지 않는다. 공식 `ConfigModule.forRoot({ isGlobal: true, ... })` 또는 모듈 import/export를 사용해 설정 초기화와 검증을 유지한다. 같은 구현을 클래스 토큰과 사용자 정의 토큰으로 노출하려면 두 번 `useClass`로 등록하지 말고 `useExisting`으로 별칭을 만든다.

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
