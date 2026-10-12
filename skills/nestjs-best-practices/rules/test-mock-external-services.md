---
title: 테스트에서 외부 서비스를 모의 객체로 대체한다
impact: HIGH
impactDescription: "빠르고 안정적이며 결정적인 테스트를 보장함"
tags: testing, mocking, external-services, jest
---

## 테스트에서 외부 서비스를 모의 객체로 대체한다

단위 테스트에서 실제 외부 서비스(API, 데이터베이스, 메시지 큐)를 호출하지 않는다. 모의 객체로 대체해 테스트를 빠르고 결정적으로 유지하고 비용이 발생하지 않도록 한다. 현실적인 모의 데이터를 사용하고 시간 초과와 오류 같은 경계 조건도 테스트한다.

**잘못된 예 (실제 API와 데이터베이스를 호출함):**

```typescript
// 테스트에서 실제 API 호출
describe('PaymentService', () => {
  it('should process payment', async () => {
    const service = new PaymentService(new StripeClient(realApiKey));
    // 실제 Stripe API에 접근!
    const result = await service.charge('tok_visa', 1000);
    // 느리고 비용이 발생하며 불안정함
  });
});

// 실제 데이터베이스 사용
describe('UsersService', () => {
  beforeEach(async () => {
    await connection.query('DELETE FROM users'); // 실제 DB를 변경함
  });

  it('should create user', async () => {
    const user = await service.create({ email: 'test@test.com' });
    // 공유 데이터베이스에 부수 효과 발생
  });
});

// 불완전한 모의 객체
const mockHttpService = {
  get: jest.fn().mockResolvedValue({ data: {} }),
  // 오류 시나리오와 다른 메서드 누락
};
```

**올바른 예 (모든 외부 의존성을 모의 객체로 대체함):**

```typescript
// HTTP 서비스를 올바르게 모의 구현
describe('WeatherService', () => {
  let service: WeatherService;
  let httpService: jest.Mocked<HttpService>;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        WeatherService,
        {
          provide: HttpService,
          useValue: {
            get: jest.fn(),
            post: jest.fn(),
          },
        },
      ],
    }).compile();

    service = module.get(WeatherService);
    httpService = module.get(HttpService);
  });

  it('should return weather data', async () => {
    const mockResponse = {
      data: { temperature: 72, humidity: 45 },
      status: 200,
      statusText: 'OK',
      headers: {},
      config: {},
    };

    httpService.get.mockReturnValue(of(mockResponse));

    const result = await service.getWeather('NYC');

    expect(result).toEqual({ temperature: 72, humidity: 45 });
  });

  it('should handle API timeout', async () => {
    httpService.get.mockReturnValue(
      throwError(() => new Error('ETIMEDOUT')),
    );

    await expect(service.getWeather('NYC')).rejects.toThrow('Weather service unavailable');
  });

  it('should handle rate limiting', async () => {
    httpService.get.mockReturnValue(
      throwError(() => ({
        response: { status: 429, data: { message: 'Rate limited' } },
      })),
    );

    await expect(service.getWeather('NYC')).rejects.toThrow(TooManyRequestsException);
  });
});

// 데이터베이스 대신 리포지토리를 모의 구현
describe('UsersService', () => {
  let service: UsersService;
  let repo: jest.Mocked<Repository<User>>;

  beforeEach(async () => {
    const mockRepo = {
      find: jest.fn(),
      findOne: jest.fn(),
      save: jest.fn(),
      delete: jest.fn(),
      createQueryBuilder: jest.fn(),
    };

    const module = await Test.createTestingModule({
      providers: [
        UsersService,
        { provide: getRepositoryToken(User), useValue: mockRepo },
      ],
    }).compile();

    service = module.get(UsersService);
    repo = module.get(getRepositoryToken(User));
  });

  it('should find user by id', async () => {
    const mockUser = { id: '1', name: 'John', email: 'john@test.com' };
    repo.findOne.mockResolvedValue(mockUser);

    const result = await service.findById('1');

    expect(result).toEqual(mockUser);
    expect(repo.findOne).toHaveBeenCalledWith({ where: { id: '1' } });
  });
});

// 복잡한 SDK를 위한 모의 객체 팩터리 생성
function createMockStripe(): jest.Mocked<Stripe> {
  return {
    paymentIntents: {
      create: jest.fn(),
      retrieve: jest.fn(),
      confirm: jest.fn(),
      cancel: jest.fn(),
    },
    customers: {
      create: jest.fn(),
      retrieve: jest.fn(),
    },
  } as any;
}

// 시간에 의존하는 테스트에서 시간 모의 처리
describe('TokenService', () => {
  beforeEach(() => {
    jest.useFakeTimers();
    jest.setSystemTime(new Date('2024-01-15'));
  });

  afterEach(() => {
    jest.useRealTimers();
  });

  it('should expire token after 1 hour', async () => {
    const token = await service.createToken();

    // 시간을 앞으로 이동
    jest.advanceTimersByTime(61 * 60 * 1000);

    expect(await service.isValid(token)).toBe(false);
  });
});
```

참고: [Jest Mocking](https://jestjs.io/docs/mock-functions)
