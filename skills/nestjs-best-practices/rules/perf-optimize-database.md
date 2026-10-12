---
title: 데이터베이스 쿼리를 최적화한다
impact: HIGH
impactDescription: "데이터베이스 쿼리는 대개 가장 큰 지연 원인임"
tags: performance, database, queries, optimization
---

## 데이터베이스 쿼리를 최적화한다

필요한 컬럼만 선택하고 적절한 인덱스를 사용하며 연관 데이터를 과도하게 조회하지 않는다. 데이터 접근 방식을 설계할 때 쿼리 성능을 고려한다. API 지연의 대부분은 비효율적인 데이터베이스 쿼리에서 비롯된다.

**잘못된 예 (데이터를 과도하게 조회하고 인덱스가 누락됨):**

```typescript
// 일부 필드만 필요해도 전체 선택
@Injectable()
export class UsersService {
  async findAllEmails(): Promise<string[]> {
    const users = await this.repo.find();
    // 모든 사용자의 모든 컬럼을 조회함
    return users.map((u) => u.email);
  }

  async getUserSummary(id: string): Promise<UserSummary> {
    const user = await this.repo.findOne({
      where: { id },
      relations: ['posts', 'posts.comments', 'posts.comments.author', 'followers'],
    });
    // 방대한 연관 관계 트리를 과도하게 조회함
    return { name: user.name, postCount: user.posts.length };
  }
}

// 자주 조회하는 컬럼에 인덱스가 없음
@Entity()
export class Order {
  @Column()
  userId: string; // 인덱스가 없어 매 조회마다 전체 테이블 스캔

  @Column()
  status: string; // 인덱스가 없어 상태 필터링이 느림
}
```

**올바른 예 (적절한 인덱스로 필요한 데이터만 조회함):**

```typescript
// 필요한 컬럼만 선택
@Injectable()
export class UsersService {
  async findAllEmails(): Promise<string[]> {
    const users = await this.repo.find({
      select: ['email'], // email 컬럼만 조회
    });
    return users.map((u) => u.email);
  }

  // 복잡한 조회에는 QueryBuilder 사용
  async getUserSummary(id: string): Promise<UserSummary> {
    return this.repo
      .createQueryBuilder('user')
      .select('user.name', 'name')
      .addSelect('COUNT(post.id)', 'postCount')
      .leftJoin('user.posts', 'post')
      .where('user.id = :id', { id })
      .groupBy('user.id')
      .getRawOne();
  }

  // 필요할 때만 연관 데이터 조회
  async getFullProfile(id: string): Promise<User> {
    return this.repo.findOne({
      where: { id },
      relations: ['posts'], // 직접 연결된 관계만 조회
      select: {
        id: true,
        name: true,
        email: true,
        posts: {
          id: true,
          title: true,
        },
      },
    });
  }
}

// 자주 조회하는 컬럼에 인덱스 추가
@Entity()
@Index(['userId'])
@Index(['status'])
@Index(['createdAt'])
@Index(['userId', 'status']) // 일반적인 쿼리 패턴을 위한 복합 인덱스
export class Order {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column()
  userId: string;

  @Column()
  status: string;

  @CreateDateColumn()
  createdAt: Date;
}

// 큰 데이터셋에는 항상 페이지네이션 적용
@Injectable()
export class OrdersService {
  async findAll(page = 1, limit = 20): Promise<PaginatedResult<Order>> {
    const [items, total] = await this.repo.findAndCount({
      skip: (page - 1) * limit,
      take: limit,
      order: { createdAt: 'DESC' },
    });

    return {
      items,
      meta: {
        page,
        limit,
        total,
        totalPages: Math.ceil(total / limit),
      },
    };
  }
}
```

참고: [TypeORM Query Builder](https://typeorm.io/select-query-builder)
