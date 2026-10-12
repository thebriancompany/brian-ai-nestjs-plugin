---
title: 데이터 접근에 리포지토리 패턴을 사용한다
impact: HIGH
impactDescription: "비즈니스 로직과 데이터베이스를 분리함"
tags: architecture, repository, data-access
---

## 데이터 접근에 리포지토리 패턴을 사용한다

복잡한 쿼리와 데이터베이스 로직을 캡슐화하는 사용자 정의 리포지토리를 만든다. 서비스가 비즈니스 로직에 집중하도록 하고, 모의 리포지토리로 테스트하기 쉽게 하며, 비즈니스 코드에 영향을 주지 않고 데이터베이스 구현을 교체할 수 있게 한다.

단순 CRUD에는 TypeORM 리포지토리를 서비스에 직접 주입해도 된다. 쿼리 재사용, 복잡한 데이터 접근 계약, 테스트 경계가 필요할 때 사용자 정의 리포지토리를 둔다. DB 교체가 자동으로 쉬워지는 것은 아니며 트랜잭션 범위도 보존해야 한다.

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

  async create(dto: CreateUserDto): Promise<User> {
    const existing = await this.usersRepo.findByEmail(dto.email);
    if (existing) {
      throw new ConflictException('Email already registered');
    }

    const user = new User();
    user.email = dto.email;
    user.name = dto.name;
    return this.usersRepo.save(user);
  }
}
```

참고: [Repository Pattern](https://martinfowler.com/eaaCatalog/repository.html)
