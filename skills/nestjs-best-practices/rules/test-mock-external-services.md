---
title: 단위 테스트에서 외부 서비스를 모의 객체로 대체한다
impact: HIGH
impactDescription: "외부 호출 없이 성공·실패 경로를 결정적으로 검증함"
tags: testing, mocking, external-services, jest
---

## 단위 테스트에서 외부 서비스를 모의 객체로 대체한다

단위 테스트는 실제 API·DB·큐 대신 소비하는 계약을 모의 구현한다. 전체 SDK를 `as any`로 위장하지 말고 사용하는 메서드만 타입으로 제한한다. 실제 DB 동작과 외부 API 계약은 별도의 격리된 통합 테스트에서 검증한다.

**잘못된 예 (비용이 발생하는 실제 호출이나 반환 타입이 다른 모의 객체):**

```typescript
const service = new PaymentService(new StripeClient(realApiKey));
await service.charge('tok_visa', 1000); // 단위 테스트가 실제 결제 API에 접근

const http = { get: jest.fn().mockResolvedValue({ data: {} }) };
// Nest HttpService.get()은 Promise가 아닌 Observable을 반환한다.
```

**올바른 예 (실제 계약과 같은 Observable을 반환함):**

```typescript
import { HttpService } from '@nestjs/axios';
import { AxiosError, AxiosHeaders, type AxiosResponse } from 'axios';
import { firstValueFrom, of, throwError } from 'rxjs';

interface Weather { temperature: number; }

@Injectable()
export class WeatherService {
  constructor(private readonly http: HttpService) {}

  async getWeather(city: string): Promise<Weather> {
    try {
      const response = await firstValueFrom(this.http.get<Weather>(
        'https://weather.example.test/current',
        { params: { city }, timeout: 2000 },
      ));
      return response.data;
    } catch (error: unknown) {
      if (error instanceof AxiosError && error.code === 'ECONNABORTED') {
        throw new GatewayTimeoutException('Weather service timed out');
      }
      throw new BadGatewayException('Weather service unavailable');
    }
  }
}

describe('WeatherService', () => {
  let moduleRef: TestingModule;
  let service: WeatherService;
  let http: jest.Mocked<Pick<HttpService, 'get'>>;

  beforeEach(async () => {
    http = { get: jest.fn() };
    moduleRef = await Test.createTestingModule({
      providers: [WeatherService, { provide: HttpService, useValue: http }],
    }).compile();
    service = moduleRef.get(WeatherService);
  });
  afterEach(async () => { await moduleRef?.close(); });

  it('returns weather data', async () => {
    const response: AxiosResponse<Weather> = {
      data: { temperature: 22 },
      status: 200,
      statusText: 'OK',
      headers: {},
      config: { headers: new AxiosHeaders() },
    };
    http.get.mockReturnValue(of(response));
    await expect(service.getWeather('Seoul')).resolves.toEqual({ temperature: 22 });
  });

  it('maps timeouts to a gateway timeout', async () => {
    http.get.mockReturnValue(throwError(() => new AxiosError('timeout', 'ECONNABORTED')));
    await expect(service.getWeather('Seoul')).rejects.toThrow(GatewayTimeoutException);
  });

  it('does not expose upstream error details', async () => {
    http.get.mockReturnValue(throwError(() => new Error('private provider details')));
    await expect(service.getWeather('Seoul')).rejects.toThrow('Weather service unavailable');
  });
});
```

서비스는 외부 응답의 런타임 스키마 검증이 필요한지 별도로 판단한다. 제네릭 타입은 실제 JSON을 검증하지 않는다. 오류 객체는 서비스가 판별하는 실제 타입·코드를 재현하고, 없는 프레임워크 예외 클래스나 문자열 하나로 모든 네트워크 오류를 표현하지 않는다.

DB 모의 객체로는 제약 조건·잠금·트랜잭션을 검증할 수 없다. 중복·동시성은 실제 DB 통합 테스트로 확인한다. 모의 객체가 리포지토리 저장 실패를 던지는 테스트도 포함해 사전 중복 조회만 믿는 구현을 통과시키지 않는다.

시간에 의존하는 테스트는 `jest.useFakeTimers()`와 `jest.setSystemTime()`을 사용하고 `afterEach`에서 `jest.useRealTimers()`로 복원한다. 타이머 콜백을 기다리는 Promise가 있다면 비동기 타이머 진행 API를 사용한다. HTTP·DB 통합 테스트에 가짜 타이머를 전역 적용하면 I/O가 멈출 수 있으므로 순수한 시간 로직으로 범위를 제한한다.

참고: [NestJS HTTP module](https://docs.nestjs.com/v11/techniques/http-module), [Jest Mock Functions](https://jestjs.io/docs/mock-functions), [Jest Timer Mocks](https://jestjs.io/docs/timer-mocks)
