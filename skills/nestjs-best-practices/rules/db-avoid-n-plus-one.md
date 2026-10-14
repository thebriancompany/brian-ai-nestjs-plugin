---
title: N+1 쿼리 문제를 피한다
impact: HIGH
impactDescription: "목록 크기에 비례하는 추가 쿼리를 줄임"
tags: database, n-plus-one, queries, performance
---

## N+1 쿼리 문제를 피한다

목록을 읽고 각 항목마다 연관 데이터를 다시 조회하면 N+1이 발생한다. 필요한 관계를 명시적으로 로딩하거나 여러 ID를 한 번에 조회한다. 조인 횟수만 줄이는 것이 목표는 아니다. 여러 일대다 관계를 조인하면 행 수와 메모리가 급증하므로 페이지 크기, 반환 컬럼, 실제 실행 계획도 확인한다.

**잘못된 예 (주문마다 추가 쿼리를 실행함):**

```typescript
async getOrdersWithItems(userId: string): Promise<Order[]> {
  const orders = await this.orderRepo.find({ where: { userId } });
  for (const order of orders) {
    order.items = await this.itemRepo.find({ where: { orderId: order.id } });
  }
  return orders;
}
// Promise.all로 바꿔도 쿼리 수는 N+1이며 연결 풀의 부하가 커질 수 있다.
```

**올바른 예 (페이지에 필요한 관계를 명시적으로 로딩함):**

```typescript
async getRecentOrdersWithItems(userId: string): Promise<Order[]> {
  return this.orderRepo.find({
    where: { userId },
    relations: { items: { product: true } },
    relationLoadStrategy: 'join',
    order: { createdAt: 'DESC', id: 'DESC' },
    take: 20,
  });
}
// join + take는 TypeORM이 ID 페이지와 관계를 별도 쿼리로 조회할 수 있다.
// 단일 SQL이라고 가정하지 말고 쿼리 로그로 확인한다.
// 한 주문의 items도 클 수 있으므로 업무상 개수 제한 또는 별도 페이지 API가 필요하다.

// 게시물이 없는 활성 사용자도 포함한다면 관계 필터를 JOIN 조건에 둔다.
async getActiveUsersWithPublishedPosts(): Promise<User[]> {
  return this.userRepo
    .createQueryBuilder('user')
    .leftJoinAndSelect('user.posts', 'post', 'post.status = :status', {
      status: 'published',
    })
    .where('user.isActive = :active', { active: true })
    .orderBy('user.id', 'ASC')
    .take(20)
    .getMany();
}
// WHERE post.status = ...로 옮기면 게시물이 없는 사용자도 결과에서 제외된다.
```

GraphQL처럼 관계 접근 시점이 나뉘면 요청마다 DataLoader를 만든다. 배치 함수는 입력 ID의 순서와 길이를 유지하고, 결과가 없는 ID에는 빈 배열을 반환한다. 다음 예제는 누구나 볼 수 있는 공개 게시물 요약만 조회한다.

```typescript
import { Injectable, Scope } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import DataLoader from 'dataloader';
import { In, Repository } from 'typeorm';

interface PostSummary {
  id: string;
  authorId: string;
  title: string;
}

@Injectable({ scope: Scope.REQUEST })
export class PublicPostsLoader {
  constructor(@InjectRepository(Post) private readonly posts: Repository<Post>) {}

  readonly byAuthor = new DataLoader<string, PostSummary[]>(async authorIds => {
    const posts = await this.posts.find({
      where: { authorId: In([...authorIds]), status: 'published' },
      select: { id: true, authorId: true, title: true },
      order: { id: 'ASC' },
    });
    const grouped = new Map<string, PostSummary[]>();
    for (const post of posts) {
      const group = grouped.get(post.authorId) ?? [];
      group.push({ id: post.id, authorId: post.authorId, title: post.title });
      grouped.set(post.authorId, group);
    }
    return authorIds.map(id => grouped.get(id) ?? []);
  }, { maxBatchSize: 100 });
}

// PublicPostsLoader를 리졸버의 모듈 providers에 등록하고 생성자로 주입한다.
@ResolveField(() => [PostSummaryDto])
posts(@Parent() user: User): Promise<PostSummary[]> {
  return this.publicPostsLoader.byAuthor.load(user.id);
}
```

공개 목록도 한 작성자의 게시물 수가 무한히 늘 수 있으면 페이지네이션이 필요하다. `maxBatchSize`는 입력 키 수만 제한한다. 비공개·테넌트 데이터는 인증된 요청의 범위를 배치 쿼리에 포함한다. 전역 싱글턴 DataLoader는 요청 간 결과와 권한을 섞을 수 있다. 같은 요청에서 변경한 데이터는 해당 키를 `clear()`하거나 `clear().prime()`한다.

TypeORM의 Promise 관계는 실제로 접근/await할 때 조회한다. 엔티티를 반환하면 언제나 직렬화가 지연 관계를 읽는다고 가정하지 말고 명시적인 DTO 변환 경로를 확인한다. `relationLoadStrategy: 'query'`도 선택할 수 있으며 어느 전략이 빠른지는 관계 크기와 DB에 따라 다르다.

검증: 1건과 다건에서 쿼리 수를 비교하고, 관계가 없는 부모의 포함 여부, DataLoader 입력 순서·중복 키·권한 범위와 최대 반환량을 확인한다. 개발용 쿼리 로그에도 민감한 파라미터가 남을 수 있으므로 운영 로깅 정책을 따른다.

참고: [TypeORM Find Options](https://typeorm.io/docs/working-with-entity-manager/find-options/), [DataLoader](https://github.com/graphql/dataloader)
