---
title: 순환 의존성을 피한다
impact: CRITICAL
impactDescription: "모듈 초기화 실패와 불명확한 생성 순서를 방지함"
tags: architecture, modules, dependencies
---

## 순환 의존성을 피한다

순환 의존성은 모듈이나 프로바이더가 서로를 직접 또는 간접적으로 참조할 때 발생한다. 먼저 두 기능을 조율하는 책임을 상위 모듈로 옮기거나, 실제로 공유하는 책임을 독립 모듈로 추출한다. 이름만 `SharedModule`로 바꾼 뒤 그 안에서 원래 두 모듈을 다시 참조하면 순환은 사라지지 않는다.

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

**올바른 예 (상위 서비스가 두 기능을 조율함):**

```typescript
// users.module.ts
@Module({
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}

// orders.module.ts
@Module({
  providers: [OrdersService],
  exports: [OrdersService],
})
export class OrdersModule {}

// user-orders.service.ts
// UsersService는 OrdersService를, OrdersService는 UsersService를 주입하지 않는다.
@Injectable()
export class UserOrdersService {
  constructor(
    private readonly users: UsersService,
    private readonly orders: OrdersService,
  ) {}

  async getForUser(userId: string) {
    const user = await this.users.findById(userId);
    if (!user) throw new NotFoundException('User not found');
    return this.orders.findByUser(user.id);
  }
}

@Module({
  imports: [UsersModule, OrdersModule],
  providers: [UserOrdersService],
  exports: [UserOrdersService],
})
export class UserOrdersModule {}
```

위 예제는 의존성 방향을 보여준다. 각 기능의 DB 등 기존 의존성 등록은 해당 모듈에 유지하고, `UsersModule`과 `OrdersModule`이 `UserOrdersModule`을 다시 가져오지 않게 한다. 생성자에서 서로의 초기화 완료에 의존하는 코드를 실행하지 않는다.

당장 구조를 바꿀 수 없다면 모듈 순환의 양쪽 `imports`에 `forwardRef(() => OtherModule)`을 적용한다. 프로바이더 생성자에도 순환이 있다면 양쪽 주입에 `@Inject(forwardRef(() => OtherService))`가 필요하다. 이는 생성 순서를 보장하지 않으며 요청 스코프 순환에는 특히 주의한다. 배럴 파일의 재수출이 만든 파일 참조 순환도 확인한다.

결과를 즉시 받아야 하는 호출을 단순히 이벤트로 바꾸면 성공·실패 계약이 달라진다. 비핵심 알림을 분리할 때만 [이벤트 규칙](arch-use-events.md)의 전달 보장과 오류 처리를 함께 적용한다.

참고: [NestJS Circular Dependency](https://docs.nestjs.com/fundamentals/circular-dependency)
