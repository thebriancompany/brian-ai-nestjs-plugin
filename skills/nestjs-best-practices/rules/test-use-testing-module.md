---
title: DI 구성을 검증할 때 TestingModule을 사용한다
impact: HIGH
impactDescription: "실제 주입 토큰과 모의 의존성으로 동작을 검증함"
tags: testing, unit-tests, mocking, jest
---

## DI 구성을 검증할 때 TestingModule을 사용한다

순수한 클래스는 `new Service(mock)`로 테스트해도 된다. `TestingModule`은 Nest의 주입 토큰·프로바이더 구성·override를 검증할 때 사용한다. 모든 테스트에 앱 전체를 import하지 말고 필요한 서비스와 모의 의존성만 등록한다.

**잘못된 예 (결과나 실패 동작 없이 호출 여부만 확인함):**

```typescript
await controller.create(dto);
expect(service.create).toHaveBeenCalled();
// 잘못된 DTO를 넘기거나 서비스 결과를 버려도 통과할 수 있다.
```

**올바른 예 (실제 DI 토큰으로 의존성을 교체하고 결과·실패를 검증함):**

```typescript
import { Test, TestingModule } from '@nestjs/testing';

interface UserSummary { id: string; name: string; }
export abstract class UserReader {
  abstract findById(id: string): Promise<UserSummary | null>;
}

@Injectable()
export class UsersService {
  constructor(private readonly reader: UserReader) {}

  async getById(id: string): Promise<UserSummary> {
    const user = await this.reader.findById(id);
    if (!user) throw new NotFoundException('User not found');
    return user;
  }
}

describe('UsersService', () => {
  let moduleRef: TestingModule;
  let service: UsersService;
  let reader: jest.Mocked<UserReader>;

  beforeEach(async () => {
    reader = { findById: jest.fn() };
    moduleRef = await Test.createTestingModule({
      providers: [UsersService, { provide: UserReader, useValue: reader }],
    }).compile();
    service = moduleRef.get(UsersService);
  });
  afterEach(async () => { await moduleRef?.close(); });

  it('returns the requested user', async () => {
    const user = { id: 'user-1', name: 'Kim' };
    reader.findById.mockResolvedValue(user);
    await expect(service.getById('user-1')).resolves.toEqual(user);
    expect(reader.findById).toHaveBeenCalledWith('user-1');
  });

  it('rejects a missing user', async () => {
    reader.findById.mockResolvedValue(null);
    await expect(service.getById('missing')).rejects.toThrow(NotFoundException);
  });

  it('preserves an infrastructure failure instead of turning it into 404', async () => {
    const failure = new Error('database unavailable');
    reader.findById.mockRejectedValue(failure);
    await expect(service.getById('user-1')).rejects.toBe(failure);
  });
});
```

기존 모듈의 프로바이더를 교체할 때는 `compile()` 전에 `overrideProvider(실제 토큰).useValue(mock)`를 호출한다.

가드에는 공개 경로·권한 없음·사용자 없음·역할 일치와 불일치를 각각 테스트한다. [가드 규칙](security-use-guards.md)의 메타데이터를 모의 처리할 때는 키에 따라 반환값을 구분한다. 모든 키에 `['admin']`을 반환하면 `isPublic`도 참이 되어 인가를 우회한 테스트가 통과한다.

```typescript
jest.spyOn(reflector, 'getAllAndOverride')
  .mockImplementation(key => key === 'roles' ? [Role.Admin] : undefined);
expect(guard.canActivate(contextWithUser({ roles: [Role.User] }))).toBe(false);
```

위의 `contextWithUser`는 테스트 전용 `ExecutionContext` 헬퍼다. 실제 클래스·핸들러 메타데이터 연결과 전역 가드 순서는 [E2E 테스트](test-e2e-supertest.md)에서 별도로 확인한다. DB 제약 조건, 쿼리 문법, 잠금은 모의 객체로 증명할 수 없으므로 실제 DB를 사용하는 격리된 통합 테스트를 추가한다. 테스트를 위해 프로덕션 코드를 불필요하게 추상화하지 않는다.

참고: [NestJS 11 Testing](https://docs.nestjs.com/v11/fundamentals/testing)
