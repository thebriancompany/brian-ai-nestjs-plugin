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
// 금액과 Account.balance는 최소 통화 단위의 안전한 JS 정수로 매핑된다는 전제다.
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
      if (!Number.isSafeInteger(source.balance) || source.balance < 0 ||
          !Number.isSafeInteger(destination.balance) || destination.balance < 0) {
        throw new Error('Invalid account balance representation');
      }
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

서비스 호출 전 인증·송금 권한을 검증한다. 권한이 계좌의 변경 가능한 상태에 달렸다면 잠근 행을 기준으로 트랜잭션 안에서 다시 확인한다. PostgreSQL `bigint`/`numeric`은 드라이버가 문자열로 반환할 수 있으므로 단순 타입 선언만으로 number 매핑을 보장하지 않는다. 큰 금액은 `bigint` 또는 정밀 십진 타입과 해당 DB 매핑을 사용하고 DB 컬럼 범위도 검증한다.

트랜잭션은 요청의 멱등성을 보장하지 않는다. 응답 유실·재시도에도 중복 송금을 막아야 하면 업무 요청 ID에 DB 유일 제약을 두고 송금과 같은 트랜잭션에서 결과를 저장한다. 데드락·직렬화 실패 재시도는 횟수를 제한하고, 트랜잭션 전체를 재실행해도 안전한 작업에만 적용한다.

TypeORM 트랜잭션 콜백에는 전달받은 `manager`로 수행하는 DB 작업만 포함한다. 외부 결제·HTTP 호출은 DB 롤백으로 취소되지 않는다. 결제 요청은 멱등 키, 주문의 대기/성공/실패 상태, 보상 처리를 설계하고 필요한 경우 outbox를 사용한다. 수동 QueryRunner가 필요하면 연결·트랜잭션 시작도 자원 해제 범위에 넣고 `finally`에서 `release()`한다.

검증: 같은 출금 계좌에 대한 동시 송금, 잔액 부족, 로그 저장 실패와 대상 계좌 미존재 시 잔액/로그가 함께 롤백되는지 실제 DB에서 확인한다. 잠금과 격리 수준은 단순 리포지토리 mock으로 검증할 수 없다.

참고: [TypeORM Transactions](https://typeorm.io/docs/advanced-topics/transactions/)
