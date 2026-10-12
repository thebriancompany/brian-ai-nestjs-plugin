---
title: 요청 횟수 제한을 구현한다
impact: HIGH
impactDescription: "남용을 막고 공정한 자원 사용을 보장함"
tags: security, rate-limiting, throttler, protection
---

## 요청 횟수 제한을 구현한다

`@nestjs/throttler`로 클라이언트별 요청 횟수를 제한한다. 인증 엔드포인트에는 엄격한 제한을, 조회 작업에는 완화된 제한을 적용하는 등 엔드포인트마다 다른 한도를 설정한다. 클러스터 배포에서는 Redis를 이용한 분산 요청 횟수 제한을 고려한다.

**잘못된 예 (민감한 엔드포인트에 요청 횟수 제한이 없음):**

```typescript
// 민감한 엔드포인트에 요청 횟수 제한이 없음
@Controller('auth')
export class AuthController {
  @Post('login')
  async login(@Body() dto: LoginDto): Promise<TokenResponse> {
    // 공격자가 무차별 대입으로 인증 정보를 추측할 수 있음
    return this.authService.login(dto);
  }

  @Post('forgot-password')
  async forgotPassword(@Body() dto: ForgotPasswordDto): Promise<void> {
    // 사용자에게 스팸 이메일을 보내는 데 악용될 수 있음
    return this.authService.sendResetEmail(dto.email);
  }
}

// 모든 엔드포인트에 같은 제한 적용
@UseGuards(ThrottlerGuard)
@Controller('api')
export class ApiController {
  @Get('public-data')
  async getPublic() {} // 더 많은 요청을 허용해야 함

  @Post('process-payment')
  async payment() {} // 더 엄격하게 제한해야 함
}
```

**올바른 예 (throttler를 설정하고 엔드포인트별 한도를 적용함):**

```typescript
import { ThrottlerModule, ThrottlerGuard, ThrottlerModuleOptions, Throttle, SkipThrottle } from '@nestjs/throttler';

// 인증 가드가 설정한 사용자 정보만 신뢰한다. JWT 검증 전에 클라이언트 헤더로 구분하지 않는다.
export function limitByUser(context: ExecutionContext): number {
  const user = context.switchToHttp().getRequest().user;
  return user ? (user.isPremium ? 1000 : 200) : 50;
}

export const throttlerOptions: ThrottlerModuleOptions = [
  { name: 'short', ttl: 1000, limit: 3 },
  { name: 'medium', ttl: 10000, limit: 20 },
  { name: 'long', ttl: 60000, limit: limitByUser },
];

@Injectable()
export class CustomThrottlerGuard extends ThrottlerGuard {
  protected async getTracker(req: Record<string, any>): Promise<string> {
    if (req.user) return `user:${req.user.id}`;
    return super.getTracker(req); // 기본 IP/IPv6 처리 유지
  }
}

@Module({
  imports: [AuthModule, ThrottlerModule.forRoot(throttlerOptions)],
  providers: [
    { provide: APP_GUARD, useClass: JwtAuthGuard },
    { provide: APP_GUARD, useClass: CustomThrottlerGuard },
  ],
})
export class AppModule {}

@Controller('auth')
export class AuthController {
  @Public()
  @Post('login')
  @Throttle({ short: { limit: 5, ttl: 60000 } })
  async login(@Body() dto: LoginDto): Promise<TokenResponse> {
    return this.authService.login(dto);
  }

  @Public()
  @Post('forgot-password')
  @Throttle({ short: { limit: 3, ttl: 3600000 } })
  async forgotPassword(@Body() dto: ForgotPasswordDto): Promise<void> {
    return this.authService.sendResetEmail(dto.email);
  }
}

@Controller('health')
export class HealthController {
  @Public()
  @Get()
  @SkipThrottle({ short: true, medium: true, long: true })
  check(): string { return 'OK'; }
}
```

`@nestjs/throttler` 6.x 기준이며 TTL 단위는 밀리초다. 각 이름의 제한은 동시에 적용되므로 long 한도를 올려도 short/medium의 순간 부하 제한은 유지된다. 전역 인증 가드는 [가드 규칙](security-use-guards.md)처럼 공개 경로를 처리해야 한다. 프록시 환경의 trust proxy는 실제 배포 경계에 맞춰 설정하고, 임의의 X-Forwarded-For를 신뢰하지 않는다.

참고: [NestJS Throttler](https://docs.nestjs.com/security/rate-limiting)
