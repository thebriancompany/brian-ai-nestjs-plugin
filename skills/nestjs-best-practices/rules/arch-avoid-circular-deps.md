---
title: 순환 의존성을 피한다
impact: CRITICAL
impactDescription: "런타임 장애의 가장 큰 원인"
tags: architecture, modules, dependencies
---

## 순환 의존성을 피한다

순환 의존성은 모듈 A가 모듈 B를 가져오고, 모듈 B도 모듈 A를 직접 또는 간접적으로 가져올 때 발생한다. NestJS는 전방 참조로 이를 해결할 수 있는 경우도 있지만, 순환 의존성은 아키텍처의 문제를 나타내므로 피해야 한다. 이는 NestJS 애플리케이션에서 런타임 장애를 일으키는 가장 큰 원인이다.

**잘못된 예 (모듈이 서로를 가져옴):**

```typescript
// users.module.ts
@Module({
  imports: [OrdersModule], // Orders는 Users가 필요하고 Users는 Orders가 필요하므로 순환함
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}

// orders.module.ts
@Module({
  imports: [UsersModule], // 순환 의존성!
  providers: [OrdersService],
  exports: [OrdersService],
})
export class OrdersModule {}
```

**올바른 예 (공통 로직을 추출하거나 이벤트를 사용함):**

```typescript
// 방법 1: 공통 로직을 제3의 모듈로 추출
// shared.module.ts
@Module({
  providers: [SharedService],
  exports: [SharedService],
})
export class SharedModule {}

// users.module.ts
@Module({
  imports: [SharedModule],
  providers: [UsersService],
})
export class UsersModule {}

// orders.module.ts
@Module({
  imports: [SharedModule],
  providers: [OrdersService],
})
export class OrdersModule {}

// 방법 2: 이벤트로 직접 결합하지 않는 통신 구현
// users.service.ts
@Injectable()
export class UsersService {
  constructor(private eventEmitter: EventEmitter2) {}

  async createUser(data: CreateUserDto) {
    const user = await this.userRepo.save(data);
    this.eventEmitter.emit('user.created', user);
    return user;
  }
}

// orders.service.ts
@Injectable()
export class OrdersService {
  @OnEvent('user.created')
  handleUserCreated(user: User) {
    // 직접 의존하지 않고 사용자 생성에 반응
  }
}
```

참고: [NestJS Circular Dependency](https://docs.nestjs.com/fundamentals/circular-dependency)
