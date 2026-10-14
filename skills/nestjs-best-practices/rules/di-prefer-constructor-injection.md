---
title: 생성자 주입을 우선한다
impact: CRITICAL
impactDescription: "의존성을 명시하고 직접 생성하는 테스트의 누락을 줄임"
tags: dependency-injection, constructor, testing
---

## 생성자 주입을 우선한다

일반적인 애플리케이션 서비스에는 생성자 주입을 우선한다. 필요한 의존성이 클래스 선언과 테스트 생성 코드에 드러난다. 기반 클래스 확장처럼 프로퍼티 주입이 유용한 경우도 있으므로 일괄 금지하지 않는다. 선택적 의존성도 생성자로 주입할 수 있다.

**잘못된 예 (프로퍼티 주입으로 필수 의존성을 숨김):**

```typescript
@Injectable()
export class UsersService {
  @Inject(USER_READER)
  private users!: UserReader;

  findById(id: string): Promise<User | null> {
    return this.users.findById(id);
  }
}

// TypeScript는 이를 허용하지만 Nest 밖에서 생성하여 주입이 수행되지 않는다.
const service = new UsersService();
await service.findById('1'); // users가 undefined이므로 실패
```

**올바른 예 (생성자 계약을 명시하고 타입 안전한 대역으로 테스트함):**

```typescript
export const USER_READER = Symbol('USER_READER');
type User = { id: string; name: string };

export interface UserReader {
  findById(id: string): Promise<User | null>;
}

@Injectable()
export class UsersService {
  constructor(@Inject(USER_READER) private readonly users: UserReader) {}

  findById(id: string): Promise<User | null> {
    return this.users.findById(id);
  }
}

it('delegates the requested ID and returns the user', async () => {
  const expected: User = { id: '1', name: 'Test' };
  const users: jest.Mocked<UserReader> = { findById: jest.fn() };
  users.findById.mockResolvedValue(expected);
  const service = new UsersService(users);

  await expect(service.findById('1')).resolves.toEqual(expected);
  expect(users.findById).toHaveBeenCalledWith('1');
});

it('preserves the missing-user contract', async () => {
  const users: UserReader = { findById: async () => null };
  await expect(new UsersService(users).findById('missing')).resolves.toBeNull();
});
```

이 단위 테스트는 서비스의 위임 계약을 검증한다. 운영 모듈에는 별도로 `USER_READER` 프로바이더를 등록하고 필요한 모듈을 가져와야 한다. 타입 단언(`as any`)으로 큰 클래스를 흉내 내기보다 서비스가 사용하는 작은 계약을 대역으로 만든다.

**선택적 의존성도 생성자에서 표현하기:**

```typescript
export const ANALYTICS = Symbol('ANALYTICS');
interface Analytics { track(event: string): Promise<void> }

@Injectable()
export class PageViewsService {
  constructor(
    @Optional() @Inject(ANALYTICS) private readonly analytics?: Analytics,
  ) {}

  async record(): Promise<void> {
    await this.analytics?.track('page_view');
  }
}
```

`@Optional()`은 프로바이더가 없어도 된다는 뜻이며 등록된 구현의 오류를 무시한다는 뜻은 아니다. 위 `record()`는 호출자가 기다리고 실패를 처리하는 계약이다. 필수 설정이나 필수 저장소의 누락을 숨기려고 `@Optional()`을 붙이지 않는다.

참고: [NestJS Providers](https://docs.nestjs.com/providers)
