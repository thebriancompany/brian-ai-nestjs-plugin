---
title: 여러 단계의 작업에 트랜잭션을 사용한다
impact: HIGH
impactDescription: "여러 단계로 이루어진 작업의 데이터 일관성을 보장함"
tags: database, transactions, typeorm, consistency
---

## 여러 단계의 작업에 트랜잭션을 사용한다

여러 데이터베이스 작업이 함께 성공하거나 함께 실패해야 한다면 트랜잭션으로 묶는다. 일부만 반영되어 데이터가 불일치하는 상황을 방지할 수 있다. 복잡한 상황에는 TypeORM의 트랜잭션 API나 DataSource의 QueryRunner를 사용한다.

**잘못된 예 (트랜잭션 없이 여러 번 저장함):**

```typescript
// 트랜잭션 없이 여러 번 저장
@Injectable()
export class OrdersService {
  async createOrder(userId: string, items: OrderItem[]): Promise<Order> {
    // 어느 단계든 실패하면 데이터가 불일치함
    const order = await this.orderRepo.save({ userId, status: 'pending' });

    for (const item of items) {
      await this.orderItemRepo.save({ orderId: order.id, ...item });
      await this.inventoryRepo.decrement({ productId: item.productId }, 'stock', item.quantity);
    }

    await this.paymentService.charge(order.id);
    // 결제가 실패해도 주문과 재고는 이미 변경된 상태!

    return order;
  }
}
```

**올바른 예 (같은 트랜잭션 매니저와 업무 조건 검증을 함께 사용함):**

```typescript
// 금액은 최소 통화 단위의 안전한 정수이며, 계좌는 같은 통화라는 전제다.
// PostgreSQL 예제. balance >= 0 등 DB 제약도 함께 유지한다.
@Injectable()
export class TransferService {
  constructor(private dataSource: DataSource) {}

  async transfer(fromId: string, toId: string, amount: number): Promise<void> {
    if (fromId === toId || !Number.isSafeInteger(amount) || amount <= 0) {
      throw new BadRequestException('Invalid transfer');
    }
    await this.dataSource.transaction(async manager => {
      // 모든 송금 경로에서 같은 순서로 잠가 교차 송금의 데드락 가능성을 줄인다.
      const accounts = new Map<string, Account>();
      for (const id of [fromId, toId].sort()) {
        const account = await manager.findOne(Account, {
          where: { id }, lock: { mode: 'pessimistic_write' },
        });
        if (!account) throw new NotFoundException('Account not found');
        accounts.set(id, account);
      }
      const source = accounts.get(fromId)!;
      const destination = accounts.get(toId)!;
      if (source.currency !== destination.currency) {
        throw new BadRequestException('Currency mismatch');
      }
      if (source.balance < amount) throw new BadRequestException('Insufficient funds');
      if (!Number.isSafeInteger(destination.balance + amount)) {
        throw new BadRequestException('Balance exceeds supported range');
      }
      const debit = await manager.decrement(Account, { id: fromId }, 'balance', amount);
      const credit = await manager.increment(Account, { id: toId }, 'balance', amount);
      if (debit.affected !== 1 || credit.affected !== 1) {
        throw new NotFoundException('Account update failed');
      }
      await manager.save(TransactionLog, { fromId, toId, amount, timestamp: new Date() });
    });
  }
}

@Injectable()
export class UsersRepository {
  constructor(private dataSource: DataSource) {}

  async createWithProfile(userData: CreateUserDto, profileData: CreateProfileDto): Promise<User> {
    return this.dataSource.transaction(async manager => {
      const user = await manager.save(User, userData);
      await manager.save(Profile, { ...profileData, userId: user.id });
      return user;
    });
  }
}
```

TypeORM 트랜잭션 콜백에는 전달받은 `manager`로 수행하는 DB 작업만 포함한다. 외부 결제·HTTP 호출은 DB 롤백으로 취소되지 않는다. 결제 요청은 멱등 키, 주문의 대기/성공/실패 상태, 보상 처리를 설계하고 필요한 경우 outbox를 사용한다. 수동 QueryRunner가 필요하면 연결·트랜잭션 시작도 자원 해제 범위에 넣고 `finally`에서 `release()`한다.

참고: [TypeORM Transactions](https://typeorm.io/transactions)
