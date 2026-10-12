---
title: 마이크로서비스에 상태 검사를 구현한다
impact: MEDIUM-HIGH
impactDescription: "오케스트레이터가 서비스 생명주기를 관리하도록 지원함"
tags: microservices, health-checks, terminus, kubernetes
---

## 마이크로서비스에 상태 검사를 구현한다

`@nestjs/terminus`로 활성 상태 검사(liveness)와 준비 상태 검사(readiness)를 구현한다. 활성 상태 검사는 서비스를 재시작해야 하는지 판단하고, 준비 상태 검사는 서비스가 트래픽을 받을 수 있는지 판단한다. 올바른 상태 검사는 Kubernetes와 로드 밸런서가 트래픽을 적절하게 라우팅하도록 돕는다.

**잘못된 예 (의존성을 확인하지 않는 단순 ping):**

```typescript
// 의존성을 확인하지 않는 단순 ping
@Controller('health')
export class HealthController {
  @Get()
  check(): string {
    return 'OK'; // 서비스가 비정상이어도 OK를 반환할 수 있음
  }
}

// 느린 의존성을 기다리는 상태 검사
@Controller('health')
export class HealthController {
  @Get()
  async check(): Promise<string> {
    // 데이터베이스가 느리면 상태 검사에서 시간 초과 발생
    await this.userRepo.findOne({ where: { id: '1' } });
    await this.redis.ping();
    await this.externalApi.healthCheck();
    return 'OK';
  }
}
```

**올바른 예 (실제 프로토콜과 제한 시간을 사용함):**

`@nestjs/terminus` 11의 `HealthIndicatorService`를 사용하는 예제다. liveness는 프로세스의 응답 여부를 확인하고, readiness는 요청 처리에 반드시 필요한 의존성과 종료 상태를 확인한다. 단순 ping 자체가 잘못된 것은 아니며, 그것만으로 DB를 사용하는 앱의 readiness를 판단하면 안 된다.

```typescript
import { HealthCheckService, HealthCheck, HealthIndicatorService,
  TypeOrmHealthIndicator } from '@nestjs/terminus';

// RedisModule이 REDIS_CLIENT 토큰으로 실제 클라이언트를 제공한다.
// 클라이언트에도 명령 시간 제한과 오프라인 큐 제한을 설정한다.
@Injectable()
export class RedisHealthIndicator {
  constructor(
    @Inject('REDIS_CLIENT') private readonly redis: { ping(): Promise<string> },
    private readonly indicators: HealthIndicatorService,
  ) {}

  async isHealthy(key: string, timeoutMs = 1000) {
    const indicator = this.indicators.check(key);
    let timer: ReturnType<typeof setTimeout> | undefined;
    try {
      const pong = await Promise.race([
        this.redis.ping(),
        new Promise<never>((_, reject) => {
          timer = setTimeout(() => reject(new Error('Redis timeout')), timeoutMs);
        }),
      ]);
      return pong === 'PONG' ? indicator.up() : indicator.down();
    } catch {
      return indicator.down();
    } finally {
      if (timer) clearTimeout(timer);
    }
  }
}

@Controller('health')
@Public()
@SkipThrottle({ short: true, medium: true, long: true })
export class HealthController {
  constructor(
    private readonly health: HealthCheckService,
    private readonly db: TypeOrmHealthIndicator,
    private readonly redis: RedisHealthIndicator,
    private readonly shutdown: ShutdownService,
  ) {}

  @Get('live')
  liveness() { return { status: 'ok' }; }

  @Get('ready')
  @HealthCheck()
  readiness() {
    if (this.shutdown.isShutdown()) {
      throw new ServiceUnavailableException('Shutting down');
    }
    return this.health.check([
      () => this.db.pingCheck('database', { timeout: 1000 }),
      () => this.redis.isHealthy('redis'), // Redis PING, HTTP가 아님
    ]);
  }
}
```

모듈에는 `TerminusModule`, 앱에서 사용하는 `TypeOrmModule`과 Redis 클라이언트 제공 모듈을 가져오고, `RedisHealthIndicator`와 컨트롤러를 등록한다. `ShutdownService`는 [종료 규칙](devops-graceful-shutdown.md)과 같은 인스턴스를 주입한다. `Public`은 [가드 규칙](security-use-guards.md)의 데코레이터이며, `SkipThrottle`의 이름은 앱의 [요청 제한 설정](security-rate-limiting.md)에 맞춘다. 전역 접두사·버전 설정을 사용한다면 실제 프로브 URL도 맞춘다.

`Promise.race`는 상태 검사 응답 시간만 제한하며 이미 전송한 Redis 명령을 취소하지 않는다. 클라이언트 자체의 명령 시간 제한도 필요하다. 외부 서비스 장애 때문에 모든 인스턴스가 재시작되지 않도록 liveness에 DB·Redis를 묶지 않는다. 선택적인 캐시를 사용한다면 Redis 장애를 readiness 실패 조건으로 삼을 필요는 없다. 상태 엔드포인트의 네트워크 접근을 제한하고 연결 정보나 오류 원문을 노출하지 않는다.

### Kubernetes 설정

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
spec:
  selector:
    matchLabels:
      app: api-service
  template:
    metadata:
      labels:
        app: api-service
    spec:
      terminationGracePeriodSeconds: 40
      containers:
        - name: api
          image: api-service:release-tag
          ports:
            - containerPort: 3000
          livenessProbe:
            httpGet:
              path: /health/live
              port: 3000
            periodSeconds: 10
            timeoutSeconds: 2
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 3000
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 1
          startupProbe:
            httpGet:
              path: /health/live
              port: 3000
            periodSeconds: 5
            failureThreshold: 30
```

참고: [NestJS Terminus](https://docs.nestjs.com/recipes/terminus)
