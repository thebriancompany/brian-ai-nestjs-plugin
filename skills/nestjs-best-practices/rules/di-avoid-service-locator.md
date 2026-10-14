---
title: 서비스 로케이터 안티패턴을 피한다
impact: HIGH
impactDescription: "의존성을 명시해 테스트와 변경 영향 분석을 쉽게 함"
tags: dependency-injection, anti-patterns, testing
---

## 서비스 로케이터 안티패턴을 피한다

이미 정해진 업무 의존성을 메서드 안에서 `ModuleRef.get()`이나 전역 컨테이너로 찾지 말고 생성자로 주입한다. `ModuleRef` 자체가 잘못된 API는 아니다. 런타임 플러그인 선택이나 동적 스코프 해석처럼 정적 주입으로 표현하기 어려운 경계에서 사용하고, 일반 서비스까지 조회 로직을 퍼뜨리지 않는다.

**잘못된 예 (업무 메서드가 의존성을 숨김):**

```typescript
@Injectable()
export class OrdersService {
  constructor(private readonly moduleRef: ModuleRef) {}

  async findForUser(userId: string): Promise<Order[]> {
    const users = this.moduleRef.get(UsersService, { strict: false });
    const orders = this.moduleRef.get(OrdersRepository, { strict: false });
    await users.requireById(userId);
    return orders.findByUser(userId);
  }
}
```

**올바른 예 (생성자로 실제 의존성을 명시함):**

```typescript
@Injectable()
export class OrdersService {
  constructor(
    private readonly users: UsersService,
    private readonly orders: OrdersRepository,
  ) {}

  async findForUser(userId: string): Promise<Order[]> {
    await this.users.requireById(userId);
    return this.orders.findByUser(userId);
  }
}

// 테스트에서는 실제 DB나 전역 컨테이너 없이 각 의존성을 대체한다.
const moduleRef = await Test.createTestingModule({
  providers: [
    OrdersService,
    { provide: UsersService, useValue: mockUsersService },
    { provide: OrdersRepository, useValue: mockOrdersRepository },
  ],
}).compile();
const service = moduleRef.get(OrdersService);
```

위 예제의 `requireById`는 없으면 예외를 던지는 사용자 조회 계약이다. 해당 서비스의 모듈은 `UsersModule`처럼 필요한 프로바이더를 내보내는 모듈을 가져와야 한다. `ModuleRef.get()`에 `{ strict: false }`를 붙여 누락된 모듈 경계를 숨기지 않는다.

**정해진 구현 중 선택할 때도 정적 주입으로 충분한 경우:**

```typescript
@Injectable()
export class HandlerSelector {
  constructor(
    private readonly email: EmailHandler,
    private readonly sms: SmsHandler,
  ) {}

  getHandler(type: 'email' | 'sms'): Handler {
    return type === 'email' ? this.email : this.sms;
  }
}
```

실제 동적 해석이 필요하다면 `get()`은 이미 등록된 정적 프로바이더를 가져오며 새 인스턴스를 만드는 메서드가 아니라는 점을 구분한다. 요청/트랜지언트 스코프는 `resolve()`로 해석한다. 같은 요청의 인스턴스를 공유해야 하면 해당 요청의 `ContextId`를 전달하고, 직접 만든 컨텍스트가 `REQUEST`에 의존한다면 `registerRequestByContextId()`로 요청 객체도 등록한다. `resolve()` 결과를 싱글턴 필드에 저장해 요청 간 공유하지 않는다.

참고: [NestJS Module Reference](https://docs.nestjs.com/fundamentals/module-ref)
