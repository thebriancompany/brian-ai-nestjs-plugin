---
title: 리스코프 치환 원칙을 지킨다
impact: HIGH
impactDescription: "호출부의 동작을 깨뜨리지 않고 구현체를 교체할 수 있게 함"
tags: dependency-injection, inheritance, solid, lsp
---

## 리스코프 치환 원칙을 지킨다

하위 타입은 프로그램의 정확성을 바꾸지 않고 기반 타입을 대체할 수 있어야 한다. NestJS의 의존성 주입에서는 인터페이스나 추상 클래스의 모든 구현체가 계약을 완전히 지켜야 한다는 뜻이다. 테스트에 사용하는 모의 결제 서비스도 실제 결제 서비스처럼 동작해야 한다. 반환값의 형태가 비슷해야 하고 오류도 같은 방식으로 처리해야 한다. 리스코프 치환 원칙을 위반하면 구현체를 교체할 때 발견하기 어려운 버그가 생긴다.

**잘못된 예 (구현체가 계약을 위반함):**

```typescript
// 계약이 명확한 기반 인터페이스
interface PaymentGateway {
  /**
   * 지정한 금액을 결제한다.
   * @returns 성공 시 PaymentResult
   * @throws 결제 실패 시 PaymentFailedException
   */
  charge(amount: number, currency: string): Promise<PaymentResult>;
}

// 운영 구현체: 계약을 따름
@Injectable()
export class StripeService implements PaymentGateway {
  async charge(amount: number, currency: string): Promise<PaymentResult> {
    const response = await this.stripe.charges.create({ amount, currency });
    return { success: true, transactionId: response.id, amount };
  }
}

// 리스코프 치환 원칙을 위반하는 모의 구현: 동작이 다름!
@Injectable()
export class MockPaymentService implements PaymentGateway {
  async charge(amount: number, currency: string): Promise<PaymentResult> {
    // 위반 1: 유효한 입력에 예외 발생 (계약은 PaymentResult 반환을 요구함)
    if (amount > 1000) {
      throw new Error('Mock does not support large amounts');
    }

    // 위반 2: PaymentResult 대신 null 반환
    if (currency !== 'USD') {
      return null as any; // 실제 서비스는 적절하게 변환하거나 거부함
    }

    // 위반 3: 필수 필드 누락
    return { success: true } as PaymentResult; // transactionId 누락!
  }
}

// 호출부는 계약을 신뢰함
@Injectable()
export class OrdersService {
  constructor(@Inject(PAYMENT_GATEWAY) private payment: PaymentGateway) {}

  async checkout(order: Order): Promise<void> {
    const result = await this.payment.charge(order.total, order.currency);
    // MockPaymentService에서 다음 코드가 실패함:
    await this.saveTransaction(result.transactionId); // undefined임!
    await this.sendReceipt(result); // null일 수 있음!
  }
}
```

**올바른 예 (구현체가 계약을 준수함):**

```typescript
// 동작을 문서화한 명확한 인터페이스
interface PaymentGateway {
  /**
   * 지정한 금액을 결제한다.
   * @param amount - 통화의 최소 단위로 표현한 금액 (센트)
   * @param currency - ISO 4217 통화 코드
   * @returns transactionId, 성공 상태, 금액을 포함하는 PaymentResult
   * @throws 결제 거절 시 PaymentFailedException
   * @throws 지원하지 않는 통화이면 InvalidCurrencyException
   */
  charge(amount: number, currency: string): Promise<PaymentResult>;

  /**
   * 이전 결제를 환불한다.
   * @throws transactionId가 유효하지 않으면 TransactionNotFoundException
   */
  refund(transactionId: string, amount?: number): Promise<RefundResult>;
}

// 운영 구현체
@Injectable()
export class StripeService implements PaymentGateway {
  async charge(amount: number, currency: string): Promise<PaymentResult> {
    try {
      const response = await this.stripe.charges.create({ amount, currency });
      return {
        success: true,
        transactionId: response.id,
        amount: response.amount,
      };
    } catch (error) {
      if (error.type === 'card_error') {
        throw new PaymentFailedException(error.message);
      }
      throw error;
    }
  }

  async refund(transactionId: string, amount?: number): Promise<RefundResult> {
    // 구현...
  }
}

// 리스코프 치환 원칙을 지키는 모의 구현: 같은 계약과 동작 형태
@Injectable()
export class MockPaymentService implements PaymentGateway {
  private transactions = new Map<string, PaymentResult>();

  async charge(amount: number, currency: string): Promise<PaymentResult> {
    // 계약 준수: 실제 서비스처럼 통화 검증
    if (!['USD', 'EUR', 'GBP'].includes(currency)) {
      throw new InvalidCurrencyException(`Unsupported currency: ${currency}`);
    }

    // 특정 테스트 시나리오를 위한 결제 거절 시뮬레이션
    if (amount === 99999) {
      throw new PaymentFailedException('Card declined (test scenario)');
    }

    // 운영 구현체와 같은 형태로 반환
    const result: PaymentResult = {
      success: true,
      transactionId: `mock_${Date.now()}_${Math.random().toString(36)}`,
      amount,
    };

    this.transactions.set(result.transactionId, result);
    return result;
  }

  async refund(transactionId: string, amount?: number): Promise<RefundResult> {
    // 계약 준수: 트랜잭션이 없으면 예외 발생
    if (!this.transactions.has(transactionId)) {
      throw new TransactionNotFoundException(transactionId);
    }

    return {
      success: true,
      refundId: `refund_${transactionId}`,
      amount: amount ?? this.transactions.get(transactionId)!.amount,
    };
  }
}

// 호출부에서 구현체를 안전하게 교체 가능
@Injectable()
export class OrdersService {
  constructor(@Inject(PAYMENT_GATEWAY) private payment: PaymentGateway) {}

  async checkout(order: Order): Promise<Order> {
    try {
      const result = await this.payment.charge(order.total, order.currency);
      // StripeService와 MockPaymentService 모두에서 동작
      order.transactionId = result.transactionId;
      order.status = 'paid';
      return order;
    } catch (error) {
      if (error instanceof PaymentFailedException) {
        order.status = 'payment_failed';
        return order;
      }
      throw error;
    }
  }
}
```

**리스코프 치환 원칙 준수 여부 테스트:**

```typescript
// 모든 구현체가 통과해야 하는 공통 테스트 모음
function testPaymentGatewayContract(
  createGateway: () => PaymentGateway,
) {
  describe('PaymentGateway contract', () => {
    let gateway: PaymentGateway;

    beforeEach(() => {
      gateway = createGateway();
    });

    it('returns PaymentResult with all required fields', async () => {
      const result = await gateway.charge(1000, 'USD');
      expect(result).toHaveProperty('success');
      expect(result).toHaveProperty('transactionId');
      expect(result).toHaveProperty('amount');
      expect(typeof result.transactionId).toBe('string');
    });

    it('throws InvalidCurrencyException for unsupported currency', async () => {
      await expect(gateway.charge(1000, 'INVALID'))
        .rejects.toThrow(InvalidCurrencyException);
    });

    it('throws TransactionNotFoundException for invalid refund', async () => {
      await expect(gateway.refund('nonexistent'))
        .rejects.toThrow(TransactionNotFoundException);
    });
  });
}

// 모든 구현체에 대해 실행
describe('StripeService', () => {
  testPaymentGatewayContract(() => new StripeService(mockStripeClient));
});

describe('MockPaymentService', () => {
  testPaymentGatewayContract(() => new MockPaymentService());
});
```

참고: [Liskov Substitution Principle](https://en.wikipedia.org/wiki/Liskov_substitution_principle)
