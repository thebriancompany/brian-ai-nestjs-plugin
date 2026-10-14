---
title: 데이터 접근에 리포지토리 패턴을 사용한다
impact: HIGH
impactDescription: "비즈니스 로직과 데이터베이스를 분리함"
tags: architecture, repository, data-access
---

## 데이터 접근에 리포지토리 패턴을 사용한다

단순 CRUD에는 TypeORM 리포지토리를 서비스에 직접 주입해도 된다. 쿼리 재사용, 복잡한 데이터 접근 계약, 테스트 경계가 필요할 때 사용자 정의 리포지토리로 데이터 접근을 캡슐화한다. 메서드 한 개를 그대로 감싸는 계층을 모든 엔티티에 추가할 필요는 없다. ORM 엔티티·쿼리 의미·트랜잭션 계약에 의존한다면 리포지토리를 만든 것만으로 DB 교체가 쉬워지지는 않는다.

**잘못된 예 (집계와 관계 선택을 잘못 결합한 복잡한 쿼리):**

```typescript
// 서비스 안의 복잡한 쿼리
@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User) private repo: Repository<User>,
  ) {}

  async findActiveWithOrders(minOrders: number): Promise<User[]> {
    // 복잡한 쿼리 로직과 비즈니스 로직이 섞여 있음
    return this.repo
      .createQueryBuilder('user')
      .leftJoinAndSelect('user.orders', 'order')
      .where('user.isActive = :active', { active: true })
      .andWhere('user.deletedAt IS NULL')
      .groupBy('user.id')
      .having('COUNT(order.id) >= :min', { min: minOrders })
      .orderBy('user.createdAt', 'DESC')
      .getMany();
  }

  // 쿼리 로직으로 서비스가 비대해짐
}
```

**올바른 예 (사용자 정의 리포지토리에 쿼리를 캡슐화함):**

```typescript
// 쿼리를 캡슐화한 사용자 정의 리포지토리
@Injectable()
export class UsersRepository {
  constructor(
    @InjectRepository(User) private repo: Repository<User>,
  ) {}

  async findById(id: string): Promise<User | null> {
    return this.repo.findOne({ where: { id } });
  }

  async findByEmail(email: string): Promise<User | null> {
    return this.repo.findOne({ where: { email } });
  }

  async findActiveWithMinOrders(minOrders: number): Promise<User[]> {
    return this.repo
      .createQueryBuilder('user')
      .leftJoinAndSelect('user.orders', 'order')
      .where('user.isActive = :active', { active: true })
      .andWhere('user.deletedAt IS NULL')
      .andWhere(qb => {
        const count = qb.subQuery()
          .select('COUNT(*)')
          .from(Order, 'countedOrder')
          .where('countedOrder.userId = user.id')
          .getQuery();
        return `${count} >= :min`;
      }, { min: minOrders })
      .orderBy('user.createdAt', 'DESC')
      .getMany();
  }

  async save(user: User): Promise<User> {
    return this.repo.save(user);
  }
}

// 비즈니스 로직만 담은 간결한 서비스
@Injectable()
export class UsersService {
  constructor(private usersRepo: UsersRepository) {}

  async getActiveUsersWithOrders(): Promise<User[]> {
    return this.usersRepo.findActiveWithMinOrders(1);
  }

  async getById(id: string): Promise<User> {
    const user = await this.usersRepo.findById(id);
    if (!user) throw new NotFoundException('User not found');
    return user;
  }
}
```

`UsersRepository`를 기능 모듈의 `providers`에 등록하고 `TypeOrmModule.forFeature([User])`로 ORM 리포지토리를 제공한다. 위 쿼리는 관계를 선택한 채 `GROUP BY user.id`로 전체 행을 집계하지 않고 별도 서브쿼리로 주문 수를 계산한다. 실제 목록 API에는 페이지 크기 제한과 정렬 기준을 추가하고, `Order.userId` 인덱스와 실행 계획을 확인한다.

트랜잭션 내부에서는 이 싱글턴이 주입받은 기본 리포지토리를 그대로 사용하지 않는다. 해당 `EntityManager`로 얻은 리포지토리를 사용하거나 명시적인 트랜잭션 문맥을 받도록 설계한다. 이메일 같은 고유성 규칙은 선조회만으로 보장되지 않으므로 DB 고유 제약과 충돌 변환도 필요하다.

참고: [NestJS TypeORM](https://docs.nestjs.com/techniques/database), [TypeORM Transactions](https://typeorm.io/docs/advanced-topics/transactions/)
