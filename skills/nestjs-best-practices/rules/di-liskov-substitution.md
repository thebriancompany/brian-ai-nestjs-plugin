---
title: 리스코프 치환 원칙을 지킨다
impact: HIGH
impactDescription: "호출부의 계약을 깨뜨리지 않고 구현체를 교체할 수 있게 함"
tags: dependency-injection, inheritance, solid, lsp
---

## 리스코프 치환 원칙을 지킨다

구현체를 교체해도 호출자가 의존하는 입력 조건, 반환값, 오류와 부수 효과의 계약은 유지되어야 한다. TypeScript의 같은 메서드 시그니처만으로 동작 호환성이 보장되지는 않는다. 테스트 대역이 외부 시스템 전체를 재현할 필요는 없지만, 테스트 대상이 의존하는 계약은 지켜야 한다.

**잘못된 예 (찾지 못하면 null이라는 계약을 예외로 바꿈):**

```typescript
interface UserReader {
  // 존재하지 않는 ID는 오류가 아니라 null이다.
  findById(id: string): Promise<UserSummary | null>;
}

class ThrowingUserReader implements UserReader {
  async findById(id: string): Promise<UserSummary> {
    throw new NotFoundException(`User ${id} not found`);
  }
}
```

**올바른 예 (반환값과 오류 계약을 명시하고 유지함):**

```typescript
export type UserSummary = { id: string; name: string };

export interface UserReader {
  /**
   * 저장소에 해당 ID가 없으면 null을 반환한다.
   * 있으면 id와 name을 포함한 새 객체를 반환한다.
   * 호출자의 반환 객체 변경은 저장소에 영향을 주지 않는다.
   * 저장소 장애는 reject하며, null로 숨기지 않는다.
   */
  findById(id: string): Promise<UserSummary | null>;
}

// 기존 외부/DB 어댑터의 읽기 경계. 모듈에서 이 추상 토큰을 등록한다.
export abstract class UserSource {
  abstract load(id: string): Promise<UserSummary | undefined>;
}

@Injectable()
export class SourceUserReader implements UserReader {
  constructor(private readonly source: UserSource) {}

  async findById(id: string): Promise<UserSummary | null> {
    const user = await this.source.load(id);
    // undefined와 null의 차이를 애플리케이션 계약으로 정규화한다.
    // source.load가 던지는 장애를 "없는 사용자"로 바꾸지 않는다.
    return user ? { id: user.id, name: user.name } : null;
  }
}

// 테스트용 인메모리 구현체: 운영 장애를 자동으로 재현하지는 않는다.
export class InMemoryUserReader implements UserReader {
  private readonly users: Map<string, UserSummary>;

  constructor(seed: UserSummary[] = []) {
    this.users = new Map(seed.map(user => [user.id, { ...user }]));
  }

  async findById(id: string): Promise<UserSummary | null> {
    const user = this.users.get(id);
    return user ? { ...user } : null;
  }
}
```

**각 구현에 적용하는 공통 계약 테스트:**

```typescript
// SourceUserReader는 외부 시스템 없이 어댑터의 계약 변환을 단위 테스트한다.
const factories: Array<{
  name: string;
  create: (seed: UserSummary[]) => UserReader;
}> = [
  { name: 'memory', create: seed => new InMemoryUserReader(seed) },
  {
    name: 'source adapter',
    create: seed => new SourceUserReader({
      load: async id => seed.find(user => user.id === id),
    }),
  },
];

describe.each(factories)('$name contract', ({ create }) => {
  let reader: UserReader;
  beforeEach(() => {
    reader = create([{ id: '1', name: 'Alice' }]);
  });

  it('returns the requested user', async () => {
    await expect(reader.findById('1')).resolves.toEqual({ id: '1', name: 'Alice' });
  });

  it('returns null for a missing ID', async () => {
    await expect(reader.findById('missing')).resolves.toBeNull();
  });

  it('does not mutate storage through a returned object', async () => {
    const user = await reader.findById('1');
    if (!user) throw new Error('Fixture missing');
    user.name = 'Changed';
    await expect(reader.findById('1')).resolves.toEqual({ id: '1', name: 'Alice' });
  });
});

it('does not turn a source failure into a missing user', async () => {
  const failure = new Error('Source unavailable');
  const reader = new SourceUserReader({ load: async () => { throw failure; } });
  await expect(reader.findById('1')).rejects.toBe(failure);
});
```

공통 계약 테스트만으로 DB 쿼리나 외부 API 규격까지 검증되는 것은 아니다. 실제 저장소 통합 테스트를 별도로 두어 오류 변환, 고유 제약, 트랜잭션 등 해당 어댑터의 책임을 검증한다.

참고: [NestJS Custom Providers](https://docs.nestjs.com/fundamentals/custom-providers), [NestJS Testing](https://docs.nestjs.com/fundamentals/testing)
