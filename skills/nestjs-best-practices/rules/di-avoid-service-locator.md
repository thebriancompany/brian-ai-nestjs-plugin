---
title: 서비스 로케이터 안티패턴을 피한다
impact: HIGH
impactDescription: "의존성을 숨기고 테스트 용이성을 해침"
tags: dependency-injection, anti-patterns, testing
---

## 서비스 로케이터 안티패턴을 피한다

런타임에 의존성을 찾기 위해 `ModuleRef.get()`이나 전역 컨테이너를 사용하는 것을 피한다. 이 방식은 의존성을 숨기고 테스트를 어렵게 하며 의존성 주입의 이점을 훼손한다. 대신 생성자 주입을 사용한다.

**잘못된 예 (서비스 로케이터 안티패턴):**

```typescript
// ModuleRef로 의존성을 동적으로 조회
@Injectable()
export class OrdersService {
  constructor(private moduleRef: ModuleRef) {}

  async createOrder(dto: CreateOrderDto): Promise<Order> {
    // 의존성이 숨겨져 생성자에서 보이지 않음
    const usersService = this.moduleRef.get(UsersService);
    const inventoryService = this.moduleRef.get(InventoryService);
    const paymentService = this.moduleRef.get(PaymentService);

    const user = await usersService.findOne(dto.userId);
    // ... 나머지 로직
  }
}

// 전역 싱글턴 컨테이너
class ServiceContainer {
  private static instance: ServiceContainer;
  private services = new Map<string, any>();

  static getInstance(): ServiceContainer {
    if (!this.instance) {
      this.instance = new ServiceContainer();
    }
    return this.instance;
  }

  get<T>(key: string): T {
    return this.services.get(key);
  }
}
```

**올바른 예 (생성자 주입으로 의존성을 명시함):**

```typescript
// 생성자 주입으로 의존성을 명시
@Injectable()
export class OrdersService {
  constructor(
    private usersService: UsersService,
    private inventoryService: InventoryService,
    private paymentService: PaymentService,
  ) {}

  async createOrder(dto: CreateOrderDto): Promise<Order> {
    const user = await this.usersService.findOne(dto.userId);
    const inventory = await this.inventoryService.check(dto.items);
    // 의존성이 명확하고 테스트 가능
  }
}

// 모의 객체로 쉽게 테스트
describe('OrdersService', () => {
  let service: OrdersService;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        OrdersService,
        { provide: UsersService, useValue: mockUsersService },
        { provide: InventoryService, useValue: mockInventoryService },
        { provide: PaymentService, useValue: mockPaymentService },
      ],
    }).compile();

    service = module.get(OrdersService);
  });
});

// 올바른 사용: 동적 인스턴스 생성을 위한 팩터리 패턴
@Injectable()
export class HandlerFactory {
  constructor(private moduleRef: ModuleRef) {}

  getHandler(type: string): Handler {
    switch (type) {
      case 'email':
        return this.moduleRef.get(EmailHandler);
      case 'sms':
        return this.moduleRef.get(SmsHandler);
      default:
        return this.moduleRef.get(DefaultHandler);
    }
  }
}
```

참고: [NestJS Module Reference](https://docs.nestjs.com/fundamentals/module-ref)
