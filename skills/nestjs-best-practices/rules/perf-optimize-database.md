---
title: 데이터베이스 쿼리를 최적화한다
impact: HIGH
impactDescription: "불필요한 조회량과 정렬·스캔 비용을 줄임"
tags: performance, database, queries, optimization
---

## 데이터베이스 쿼리를 최적화한다

느린 요청의 쿼리 수, 반환량, 실행 계획을 먼저 측정한다. 필요한 컬럼과 관계만 조회하고 필터·정렬 패턴에 맞는 인덱스를 선택한다. 모든 지연이 DB에서 온다고 단정하거나 모든 컬럼에 인덱스를 붙이지 않는다. 인덱스는 저장 공간과 쓰기 비용을 늘린다.

**잘못된 예 (집계 하나를 위해 관계 전체를 로딩함):**

```typescript
async getUserSummary(id: string): Promise<UserSummary> {
  const user = await this.repo.findOne({
    where: { id },
    relations: ['posts', 'posts.comments', 'posts.comments.author', 'followers'],
  });
  return { name: user.name, postCount: user.posts.length };
  // 존재하지 않는 사용자도 처리하지 않음
}
```

**올바른 예 (집계 결과를 명시적인 DTO로 변환함):**

```typescript
interface UserSummary {
  name: string;
  postCount: number;
}

// PostgreSQL + pg 드라이버의 COUNT는 bigint이며 기본적으로 문자열로 반환된다.
async getUserSummary(id: string): Promise<UserSummary> {
  const row = await this.repo
    .createQueryBuilder('user')
    .select('user.name', 'name')
    .addSelect('COUNT(post.id)', 'postCount')
    .leftJoin('user.posts', 'post')
    .where('user.id = :id', { id })
    .groupBy('user.id')
    .addGroupBy('user.name')
    .getRawOne<{ name: string; postCount: string }>();

  if (!row) throw new NotFoundException('User not found');
  const postCount = Number(row.postCount);
  if (!Number.isSafeInteger(postCount) || postCount < 0) {
    throw new Error('Post count exceeds the numeric API range');
  }
  return { name: row.name, postCount };
}
// raw 결과에는 엔티티 변환기/직렬화 데코레이터가 자동 적용되지 않는다.
// 더 큰 정수가 필요하면 API에서 숫자 대신 십진 문자열을 사용한다.
```

목록 API는 크기를 제한하고 고유한 보조 정렬 키를 둔다. 작은 목록의 offset 페이지네이션은 충분히 유효하다. 깊은 페이지에서 비용이 커지면 커서 방식으로 전환하고, 전체 개수가 필요 없으면 매번 `COUNT(*)`를 실행하지 않는다.

```typescript
@Injectable()
export class OrdersService {
  constructor(@InjectRepository(Order) private readonly repo: Repository<Order>) {}

  // userId는 인증된 사용자 범위에서 전달한다.
  async findForUser(userId: string, page = 1, limit = 20) {
    if (!Number.isSafeInteger(page) || page < 1 ||
        !Number.isSafeInteger(limit) || limit < 1 || limit > 100) {
      throw new BadRequestException('Invalid pagination');
    }
    const skip = (page - 1) * limit;
    if (!Number.isSafeInteger(skip) || skip > 10_000) {
      throw new BadRequestException('Pagination window exceeded');
    }
    const [items, total] = await this.repo.findAndCount({
      where: { userId },
      select: { id: true, status: true, createdAt: true },
      skip,
      take: limit,
      order: { createdAt: 'DESC', id: 'DESC' },
    });
    return {
      items: items.map(({ id, status, createdAt }) => ({ id, status, createdAt })),
      meta: { page, limit, total, totalPages: Math.ceil(total / limit) },
    };
  }
}

// 위 WHERE userId + ORDER BY createdAt, id 패턴의 인덱스 후보.
// 마이그레이션으로 적용하고 대표적인 데이터 분포에서 계획을 확인한다.
@Entity()
@Index(['userId', 'createdAt', 'id'])
export class Order {
  @PrimaryGeneratedColumn('uuid')
  id!: string;

  @Column('uuid')
  userId!: string;

  @Column()
  status!: string;

  @CreateDateColumn()
  createdAt!: Date;
}
```

`findAndCount`의 행과 개수는 동시 변경 아래에서 같은 스냅샷을 보장하지 않는다. 정확한 일관성이 필요한 보고서만 적절한 격리 수준을 사용한다. 고유한 정렬도 페이지 사이 삽입/삭제에 의한 offset 이동을 막지는 못한다. 커서는 정렬에 쓰는 모든 키를 포함하고 정렬 방향에 맞는 비교 조건을 사용한다. 전체 이메일 내보내기 같은 작업은 모든 행을 메모리에 올리는 대신 배치/스트리밍으로 처리한다.

검증: 빈 결과와 미존재 단건, 동일한 생성 시각, 잘못된/과도한 페이지 입력을 확인한다. 운영 크기에 가까운 데이터로 인덱스 전후 실행 계획과 지연을 비교한다. `EXPLAIN ANALYZE`는 쿼리를 실제 실행하므로 변경 쿼리에는 주의한다.

참고: [TypeORM Query Builder](https://typeorm.io/docs/query-builder/select-query-builder/), [PostgreSQL Aggregate Functions](https://www.postgresql.org/docs/current/functions-aggregate.html), [node-postgres Data Types](https://node-postgres.com/features/types)
