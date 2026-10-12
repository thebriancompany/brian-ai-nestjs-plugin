---
title: 단위 테스트에 TestingModule을 사용한다
impact: HIGH
impactDescription: "모의 의존성으로 올바르게 격리된 테스트를 수행함"
tags: testing, unit-tests, mocking, jest
---

## 단위 테스트에 TestingModule을 사용한다

`@nestjs/testing` 모듈로 모의 의존성을 사용하는 격리된 테스트 환경을 만든다. 이렇게 하면 테스트가 빠르게 실행되고 외부 서비스에 의존하지 않으며 비즈니스 로직을 독립적으로 검증할 수 있다.

모의 의존성을 넣어 `new UsersService(mockRepo)`로 생성하는 단위 테스트도 유효하다. `TestingModule`은 Nest의 프로바이더 등록, 토큰, override 동작까지 검증할 때 사용한다.

**잘못된 예 (단위 테스트에서 실제 외부 의존성을 사용함):**

```typescript
// DI 없이 서비스 인스턴스를 직접 생성
describe('UsersService', () => {
  it('should create user', async () => {
    // 문제는 new 자체가 아니라 실제 DB 의존성을 사용한다는 점
    const repo = new UserRepository(); // 실제 리포지토리!
    const service = new UsersService(repo);

    const user = await service.create({ name: 'Test' });
    // 실제 데이터베이스에 접근함!
  });
});

// 구현 세부 사항 테스트
describe('UsersController', () => {
  it('should call service', async () => {
    const service = { create: jest.fn() };
    const controller = new UsersController(service as any);

    await controller.create({ name: 'Test' });

    expect(service.create).toHaveBeenCalled(); // 동작 대신 구현을 테스트함
  });
});
```

**올바른 예 (Test.createTestingModule과 모의 의존성을 사용함):**

```typescript
// 올바른 DI를 위해 Test.createTestingModule 사용
import { Test, TestingModule } from '@nestjs/testing';

describe('UsersService', () => {
  let service: UsersService;
  let repo: jest.Mocked<UserRepository>;
  let module: TestingModule;

  beforeEach(async () => {
    module = await Test.createTestingModule({
      providers: [
        UsersService,
        {
          provide: UserRepository,
          useValue: {
            save: jest.fn(),
            findOne: jest.fn(),
            find: jest.fn(),
          },
        },
      ],
    }).compile();

    service = module.get<UsersService>(UsersService);
    repo = module.get(UserRepository);
  });

  afterEach(async () => {
    await module.close();
    jest.clearAllMocks();
  });

  describe('create', () => {
    it('should save and return user', async () => {
      const dto = { name: 'John', email: 'john@test.com' };
      const expectedUser = { id: '1', ...dto };

      repo.save.mockResolvedValue(expectedUser);

      const result = await service.create(dto);

      expect(result).toEqual(expectedUser);
      expect(repo.save).toHaveBeenCalledWith(dto);
    });

    it('should throw on duplicate email', async () => {
      repo.findOne.mockResolvedValue({ id: '1', email: 'test@test.com' });

      await expect(
        service.create({ name: 'Test', email: 'test@test.com' }),
      ).rejects.toThrow(ConflictException);
    });
  });

  describe('findById', () => {
    it('should return user when found', async () => {
      const user = { id: '1', name: 'John' };
      repo.findOne.mockResolvedValue(user);

      const result = await service.findById('1');

      expect(result).toEqual(user);
    });

    it('should throw NotFoundException when not found', async () => {
      repo.findOne.mockResolvedValue(null);

      await expect(service.findById('999')).rejects.toThrow(NotFoundException);
    });
  });
});

// 가드와 인터셉터 테스트
describe('RolesGuard', () => {
  let guard: RolesGuard;
  let reflector: Reflector;
  let module: TestingModule;

  beforeEach(async () => {
    module = await Test.createTestingModule({
      providers: [RolesGuard, Reflector],
    }).compile();

    guard = module.get<RolesGuard>(RolesGuard);
    reflector = module.get<Reflector>(Reflector);
  });

  afterEach(async () => { await module.close(); });

  it('should allow when no roles required', () => {
    const context = createMockExecutionContext({ user: { roles: [] } });
    jest.spyOn(reflector, 'getAllAndOverride').mockReturnValue(undefined);

    expect(guard.canActivate(context)).toBe(true);
  });

  it('should allow admin for admin-only route', () => {
    const context = createMockExecutionContext({ user: { roles: ['admin'] } });
    jest.spyOn(reflector, 'getAllAndOverride')
      .mockImplementation(key => key === 'roles' ? ['admin'] : undefined);

    expect(guard.canActivate(context)).toBe(true);
  });
});

function createMockExecutionContext(request: Partial<Request>): ExecutionContext {
  return {
    switchToHttp: () => ({
      getRequest: () => request,
    }),
    getHandler: () => jest.fn(),
    getClass: () => jest.fn(),
  } as ExecutionContext;
}
```

참고: [NestJS Testing](https://docs.nestjs.com/fundamentals/testing)
