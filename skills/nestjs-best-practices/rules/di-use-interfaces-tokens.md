---
title: 인터페이스에 주입 토큰을 사용한다
impact: HIGH
impactDescription: "런타임에 인터페이스 기반 의존성 주입을 가능하게 함"
tags: dependency-injection, tokens, interfaces
---

## 인터페이스에 주입 토큰을 사용한다

TypeScript 인터페이스는 컴파일 시 사라지므로 주입 토큰으로 사용할 수 없다. 인터페이스의 구현체를 주입하려면 문자열 토큰, 심벌, 추상 클래스를 사용한다. 이렇게 하면 테스트나 실행 환경에 맞춰 구현체를 교체할 수 있다.

**잘못된 예 (인터페이스를 토큰으로 사용하려 함):**

```typescript
// 인터페이스는 주입 토큰으로 사용할 수 없음
interface PaymentGateway {
  charge(amount: number): Promise<PaymentResult>;
}

@Injectable()
export class StripeService implements PaymentGateway {
  charge(amount: number) { /* ... */ }
}

@Injectable()
export class OrdersService {
  // 동작하지 않음: PaymentGateway는 런타임에 존재하지 않음
  constructor(private payment: PaymentGateway) {}
}
```

**올바른 예 (심벌 토큰이나 추상 클래스를 사용함):**

```typescript
// 방법 1: 문자열/심벌 토큰 (가장 유연함)
export const PAYMENT_GATEWAY = Symbol('PAYMENT_GATEWAY');

export interface PaymentGateway {
  charge(amount: number): Promise<PaymentResult>;
}

@Injectable()
export class StripeService implements PaymentGateway {
  async charge(amount: number): Promise<PaymentResult> {
    // Stripe 구현
  }
}

@Injectable()
export class MockPaymentService implements PaymentGateway {
  async charge(amount: number): Promise<PaymentResult> {
    return { success: true, id: 'mock-id' };
  }
}

// 모듈 등록
@Module({
  providers: [
    {
      provide: PAYMENT_GATEWAY,
      useClass: process.env.NODE_ENV === 'test'
        ? MockPaymentService
        : StripeService,
    },
  ],
  exports: [PAYMENT_GATEWAY],
})
export class PaymentModule {}

// 주입
@Injectable()
export class OrdersService {
  constructor(
    @Inject(PAYMENT_GATEWAY) private payment: PaymentGateway,
  ) {}

  async createOrder(dto: CreateOrderDto) {
    await this.payment.charge(dto.amount);
  }
}

// 방법 2: 추상 클래스 (런타임 타입 정보를 가짐)
export abstract class PaymentGateway {
  abstract charge(amount: number): Promise<PaymentResult>;
}

@Injectable()
export class StripeService extends PaymentGateway {
  async charge(amount: number): Promise<PaymentResult> {
    // 구현
  }
}

// 추상 클래스에는 @Inject가 필요 없음
@Injectable()
export class OrdersService {
  constructor(private payment: PaymentGateway) {}
}
```

참고: [NestJS Custom Providers](https://docs.nestjs.com/fundamentals/custom-providers)
