---
title: N+1 쿼리 문제를 피한다
impact: HIGH
impactDescription: "N+1 쿼리는 가장 흔한 성능 저하 원인 중 하나임"
tags: database, n-plus-one, queries, performance
---

## N+1 쿼리 문제를 피한다

N+1 쿼리는 엔티티 목록을 조회한 뒤, 연관 데이터를 불러오려고 각 엔티티마다 추가 쿼리를 실행할 때 발생한다. `relations`를 통한 즉시 로딩, 쿼리 빌더의 조인, DataLoader를 활용해 여러 조회를 효율적으로 묶는다.

**잘못된 예 (반복문 안의 지연 로딩으로 N+1이 발생함):**

```typescript
// 반복문 안의 지연 로딩으로 N+1 발생
@Injectable()
export class OrdersService {
  async getOrdersWithItems(userId: string): Promise<Order[]> {
    const orders = await this.orderRepo.find({ where: { userId } });
    // 주문 조회 쿼리 1회

    for (const order of orders) {
      // 주문마다 1회씩 추가 쿼리 N회!
      order.items = await this.itemRepo.find({ where: { orderId: order.id } });
    }

    return orders;
  }
}

// 미리 불러오지 않고 지연 로딩 관계에 접근
@Controller('users')
export class UsersController {
  @Get()
  async findAll(): Promise<User[]> {
    const users = await this.userRepo.find();
    // User.posts가 지연 로딩되면 직렬화 시 N개의 쿼리가 실행됨
    return users; // user.posts에 접근할 때마다 쿼리 1회
  }
}
```

**올바른 예 (relations로 연관 데이터를 즉시 로딩함):**

```typescript
// relations 옵션으로 즉시 로딩
@Injectable()
export class OrdersService {
  async getOrdersWithItems(userId: string): Promise<Order[]> {
    // JOIN을 사용하는 단일 쿼리
    return this.orderRepo.find({
      where: { userId },
      relations: ['items', 'items.product'],
    });
  }
}

// 복잡한 조인에는 QueryBuilder 사용
@Injectable()
export class UsersService {
  async getUsersWithPostCounts(): Promise<UserWithPostCount[]> {
    return this.userRepo
      .createQueryBuilder('user')
      .leftJoin('user.posts', 'post')
      .select('user.id', 'id')
      .addSelect('user.name', 'name')
      .addSelect('COUNT(post.id)', 'postCount')
      .groupBy('user.id')
      .getRawMany();
  }

  async getActiveUsersWithPosts(): Promise<User[]> {
    return this.userRepo
      .createQueryBuilder('user')
      .leftJoinAndSelect('user.posts', 'post')
      .leftJoinAndSelect('post.comments', 'comment')
      .where('user.isActive = :active', { active: true })
      .andWhere('post.status = :status', { status: 'published' })
      .getMany();
  }
}

// find 옵션으로 특정 필드 선택
async getOrderSummaries(userId: string): Promise<OrderSummary[]> {
  return this.orderRepo.find({
    where: { userId },
    relations: ['items'],
    select: {
      id: true,
      total: true,
      status: true,
      items: {
        id: true,
        quantity: true,
        price: true,
      },
    },
  });
}

// GraphQL에서 DataLoader로 쿼리를 묶고 캐싱
import DataLoader from 'dataloader';

@Injectable({ scope: Scope.REQUEST })
export class PostsLoader {
  constructor(private postsService: PostsService) {}

  readonly batchPosts = new DataLoader<string, Post[]>(async (userIds) => {
    // 모든 사용자의 게시물을 단일 쿼리로 조회
    const posts = await this.postsService.findByUserIds([...userIds]);

    // userId별로 묶기
    const postsMap = new Map<string, Post[]>();
    for (const post of posts) {
      const userPosts = postsMap.get(post.userId) || [];
      userPosts.push(post);
      postsMap.set(post.userId, userPosts);
    }

    // 입력과 같은 순서로 반환
    return userIds.map((id) => postsMap.get(id) || []);
  });
}

// 리졸버에서 사용
@ResolveField()
async posts(@Parent() user: User): Promise<Post[]> {
  // DataLoader가 여러 호출을 하나의 쿼리로 묶음
  return this.postsLoader.batchPosts.load(user.id);
}

// 개발 환경에서 쿼리 로깅을 활성화해 N+1 탐지
TypeOrmModule.forRoot({
  logging: ['query', 'error'],
  logger: 'advanced-console',
});
```

참고: [TypeORM Relations](https://typeorm.io/relations)
