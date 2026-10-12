---
title: 생성자 주입을 우선한다
impact: CRITICAL
impactDescription: "올바른 의존성 주입과 테스트에 필요함"
tags: dependency-injection, constructor, testing
---

## 생성자 주입을 우선한다

일반적인 애플리케이션 서비스에는 생성자 주입을 우선한다. 기존 기반 클래스나 프레임워크 확장처럼 프로퍼티 주입이 필요한 경우에는 그 제약을 확인한다. 생성자 주입은 의존성을 명시하고 TypeScript 타입 검사를 활용하게 하며, 클래스 인스턴스를 생성할 때 의존성이 준비되도록 보장하고 테스트 용이성을 높인다. 올바른 의존성 주입, 테스트, TypeScript 지원에 필요한 방식이다.

**잘못된 예 (프로퍼티 주입으로 의존성을 숨김):**

```typescript
// 프로퍼티 주입: 필요하지 않으면 피함
@Injectable()
export class UsersService {
  @Inject()
  private userRepo: UserRepository; // 숨겨진 의존성

  @Inject('CONFIG')
  private config: ConfigType; // 이 의존성도 숨겨짐

  async findAll() {
    return this.userRepo.find();
  }
}

// 문제점:
// 1. 생성자에서 의존성을 볼 수 없음
// 2. 테스트에서 의존성 없이 서비스 인스턴스를 생성할 수 있음
// 3. 인스턴스 생성 시 TypeScript가 의존성 타입을 강제할 수 없음
```

**올바른 예 (생성자 주입으로 의존성을 명시함):**

```typescript
// 생성자 주입: 명시적이며 테스트 가능
@Injectable()
export class UsersService {
  constructor(
    private readonly userRepo: UserRepository,
    @Inject('CONFIG') private readonly config: ConfigType,
  ) {}

  async findAll(): Promise<User[]> {
    return this.userRepo.find();
  }
}

// 테스트가 간단함
describe('UsersService', () => {
  let service: UsersService;
  let mockRepo: jest.Mocked<UserRepository>;

  beforeEach(() => {
    mockRepo = {
      find: jest.fn(),
      save: jest.fn(),
    } as any;

    service = new UsersService(mockRepo, { dbUrl: 'test' });
  });

  it('should find all users', async () => {
    mockRepo.find.mockResolvedValue([{ id: '1', name: 'Test' }]);
    const result = await service.findAll();
    expect(result).toHaveLength(1);
  });
});

// 선택적 의존성에만 프로퍼티 주입 사용
@Injectable()
export class LoggingService {
  @Optional()
  @Inject('ANALYTICS')
  private analytics?: AnalyticsService;

  log(message: string) {
    console.log(message);
    this.analytics?.track('log', message); // 선택적인 부가 기능
  }
}
```

참고: [NestJS Providers](https://docs.nestjs.com/providers)
