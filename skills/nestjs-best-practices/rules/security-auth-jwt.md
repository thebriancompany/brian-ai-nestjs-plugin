---
title: 안전한 JWT 인증을 구현한다
impact: CRITICAL
impactDescription: "안전한 API에 필수적임"
tags: security, jwt, authentication, tokens
---

## 안전한 JWT 인증을 구현한다

인증에 `@nestjs/jwt`와 `@nestjs/passport`를 함께 사용한다. 비밀 값을 안전하게 저장하고, 토큰의 수명을 적절하게 설정하며, 리프레시 토큰을 구현하고, 토큰을 올바르게 검증한다. JWT 페이로드에 민감한 데이터를 노출하지 않는다.

**잘못된 예 (안전하지 않은 JWT 구현):**

```typescript
// 비밀 값을 코드에 하드코딩
@Module({
  imports: [
    JwtModule.register({
      secret: 'my-secret-key', // 코드에 노출됨
      signOptions: { expiresIn: '7d' }, // 너무 김
    }),
  ],
})
export class AuthModule {}

// JWT에 민감한 데이터 저장
async login(user: User): Promise<{ accessToken: string }> {
  const payload = {
    sub: user.id,
    email: user.email,
    password: user.password, // 비밀번호를 절대 포함하지 말 것!
    ssn: user.ssn, // 민감한 데이터를 절대 포함하지 말 것!
    isAdmin: user.isAdmin, // 검증하지 않으면 변조될 수 있음
  };
  return { accessToken: this.jwtService.sign(payload) };
}

// 토큰 검증 생략
@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor() {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      secretOrKey: 'my-secret',
    });
  }

  async validate(payload: any): Promise<any> {
    return payload; // 사용자 존재 여부를 검증하지 않음
  }
}
```

**올바른 예 (리프레시 토큰을 포함한 안전한 JWT 구현):**

```typescript
// 안전한 JWT 설정
@Module({
  imports: [
    JwtModule.registerAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        secret: config.get<string>('JWT_SECRET'),
        signOptions: {
          expiresIn: '15m', // 수명이 짧은 액세스 토큰
          issuer: config.get<string>('JWT_ISSUER'),
          audience: config.get<string>('JWT_AUDIENCE'),
        },
      }),
    }),
    PassportModule.register({ defaultStrategy: 'jwt' }),
  ],
})
export class AuthModule {}

// 최소한의 JWT 페이로드
@Injectable()
export class AuthService {
  async login(user: User): Promise<TokenResponse> {
    // 필요한 비민감 데이터만 포함
    const payload: JwtPayload = {
      sub: user.id,
      email: user.email,
      roles: user.roles,
      iat: Math.floor(Date.now() / 1000),
    };

    const accessToken = this.jwtService.sign(payload);
    const refreshToken = await this.createRefreshToken(user.id);

    return { accessToken, refreshToken, expiresIn: 900 };
  }

  private async createRefreshToken(userId: string): Promise<string> {
    const token = randomBytes(32).toString('hex');
    const hashedToken = await bcrypt.hash(token, 10);

    await this.refreshTokenRepo.save({
      userId,
      token: hashedToken,
      expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000), // 7일
    });

    return token;
  }
}

// 검증을 수행하는 올바른 JWT 전략
@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(
    private config: ConfigService,
    private usersService: UsersService,
  ) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      secretOrKey: config.get<string>('JWT_SECRET'),
      ignoreExpiration: false,
      issuer: config.get<string>('JWT_ISSUER'),
      audience: config.get<string>('JWT_AUDIENCE'),
    });
  }

  async validate(payload: JwtPayload): Promise<User> {
    // 사용자가 여전히 존재하고 활성 상태인지 확인
    const user = await this.usersService.findById(payload.sub);

    if (!user || !user.isActive) {
      throw new UnauthorizedException('User not found or inactive');
    }

    // 비밀번호 변경 전에 발급된 토큰인지 확인
    if (user.passwordChangedAt) {
      const tokenIssuedAt = new Date(payload.iat * 1000);
      if (tokenIssuedAt < user.passwordChangedAt) {
        throw new UnauthorizedException('Token invalidated by password change');
      }
    }

    return user;
  }
}
```

참고: [NestJS Authentication](https://docs.nestjs.com/security/authentication)
