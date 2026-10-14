---
title: 요청 횟수 제한을 구현한다
impact: HIGH
impactDescription: "남용을 막고 공정한 자원 사용을 보장함"
tags: security, rate-limiting, throttler, protection
---

## 요청 횟수 제한을 구현한다

`@nestjs/throttler`로 클라이언트별 요청 횟수를 제한한다. 인증 엔드포인트에는 엄격한 제한을, 조회 작업에는 완화된 제한을 적용하는 등 엔드포인트마다 다른 한도를 설정한다. 여러 인스턴스에서 한도를 공유하려면 Redis 등의 공용 저장소를 연결한다. 기본 메모리 저장소의 카운터는 프로세스별로 분리된다.

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

export const throttlerOptions: ThrottlerModuleOptions = [
  { name: 'short', ttl: 1000, limit: 3 },
  { name: 'medium', ttl: 10000, limit: 20 },
  { name: 'long', ttl: 60000, limit: 100 },
];

@Module({
  imports: [AuthModule, ThrottlerModule.forRoot(throttlerOptions)],
  providers: [
    // IP 기준 제한을 먼저 적용해 잘못된 JWT 요청도 검증 비용을 소모하기 전에 제한한다.
    { provide: APP_GUARD, useClass: ThrottlerGuard },
    { provide: APP_GUARD, useClass: JwtAuthGuard },
    { provide: APP_GUARD, useClass: RolesGuard },
  ],
})
export class AppModule {}

@Controller('auth')
export class AuthController {
  constructor(private readonly authService: AuthService) {}

  @Public()
  @Post('login')
  @Throttle({ long: { limit: 5, ttl: 60000 } })
  async login(@Body() dto: LoginDto): Promise<TokenResponse> {
    return this.authService.login(dto);
  }

  @Public()
  @Post('forgot-password')
  @Throttle({ long: { limit: 3, ttl: 3600000 } })
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

`@nestjs/throttler` 6.x 기준이며 TTL 단위는 밀리초다. 각 이름의 제한은 동시에 적용되므로 long 설정을 재정의해도 short/medium의 순간 부하 제한은 유지된다. 기본 키는 IP와 핸들러를 기준으로 하므로 이 예제는 전체 API 호출량을 합산한 사용자별 할당량이 아니다. 전역 인증 가드는 [가드 규칙](security-use-guards.md)처럼 공개 경로를 처리해야 한다. 프록시 환경의 trust proxy는 실제 배포 경계에 맞춰 설정하고, 임의의 X-Forwarded-For를 신뢰하지 않는다.

사용자별 한도가 추가로 필요하면 인증 이후 현재 `request.user.id`로 별도 제한을 적용한다. 로그인·비밀번호 재설정에는 IP 제한만으로 분산 공격을 막을 수 없으므로 정규화한 계정 식별자별 한도도 함께 둔다. 요청 본문의 사용자 ID나 검증 전 토큰 클레임을 인증된 사용자로 취급하지 않는다. 공개 상태 확인의 제한 제외는 값싼 liveness 검사에만 적용하고, 저장소·외부 서비스 장애 시 한도 처리 정책도 결정한다.

참고: [NestJS Throttler](https://docs.nestjs.com/security/rate-limiting)
