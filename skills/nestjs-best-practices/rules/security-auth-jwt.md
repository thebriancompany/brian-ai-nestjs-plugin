---
title: 안전한 JWT 인증을 구현한다
impact: CRITICAL
impactDescription: "서명 검증과 현재 계정 상태 확인으로 접근을 제어함"
tags: security, jwt, authentication, tokens
---

## 안전한 JWT 인증을 구현한다

JWT를 사용하는 프로젝트에서는 서명·알고리즘·발급자·대상·만료와 애플리케이션의 필수 클레임을 함께 검증한다. 서명된 토큰도 계정 비활성화나 권한 변경 이후에는 오래된 정보를 담고 있다. 요청마다 현재 계정 상태를 조회하고, 비밀번호 변경·전체 로그아웃 등에는 서버의 토큰 버전을 증가시켜 기존 토큰을 무효화한다. JWT 페이로드는 암호화되지 않으므로 비밀번호나 민감한 정보를 담지 않는다.

**잘못된 예 (서명 검증만으로 현재 사용자를 신뢰함):**

```typescript
JwtModule.register({
  secret: 'my-secret-key', // 저장소에 노출된 비밀 값
  signOptions: { expiresIn: '7d' },
});

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor() {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      secretOrKey: 'my-secret-key',
    });
  }

  async validate(payload: any) {
    // 서명과 만료는 검사하지만, 삭제·비활성화·권한 변경을 반영하지 못한다.
    return payload;
  }
}
```

**올바른 예 (필수 클레임과 현재 계정을 검증함):**

아래 예제는 NestJS 11의 `@nestjs/jwt`, `@nestjs/passport`, `passport-jwt` 조합이다. `UsersModule`은 `UsersService`를 내보내며, `findAuthUserById()`는 계정이 없으면 `null`을 반환한다. `id`는 이 프로젝트에서 UUID v4이고 `tokenVersion`은 DB의 음수가 아닌 정수다. 다른 ID 형식을 쓰면 검증도 함께 변경한다.

```typescript
import { isUUID } from 'class-validator';
import { ExtractJwt, Strategy } from 'passport-jwt';
import { PassportStrategy } from '@nestjs/passport';

interface AccessTokenClaims {
  sub: string;
  tokenUse: 'access';
  tokenVersion: number;
  iat: number;
  exp: number;
}

export interface AuthenticatedUser {
  id: string;
  roles: Role[];
}

function isAccessTokenClaims(value: unknown): value is AccessTokenClaims {
  if (typeof value !== 'object' || value === null) return false;
  const claims = value as Record<string, unknown>;
  return (
    typeof claims.sub === 'string' && isUUID(claims.sub, '4') &&
    claims.tokenUse === 'access' &&
    typeof claims.tokenVersion === 'number' &&
    Number.isSafeInteger(claims.tokenVersion) && claims.tokenVersion >= 0 &&
    typeof claims.iat === 'number' && Number.isSafeInteger(claims.iat) &&
    claims.iat > 0 && claims.iat <= Math.floor(Date.now() / 1000) &&
    typeof claims.exp === 'number' && Number.isSafeInteger(claims.exp) &&
    claims.exp > claims.iat && claims.exp - claims.iat <= 900
  );
}

@Injectable()
export class AuthService {
  constructor(private readonly jwtService: JwtService) {}

  // 자격 증명 검증을 마친 서버 측 사용자만 전달한다. 요청 본문을 넘기지 않는다.
  async issueAccessToken(user: User) {
    if (!user.isActive) throw new UnauthorizedException();
    const accessToken = await this.jwtService.signAsync({
      sub: user.id,
      tokenUse: 'access',
      tokenVersion: user.tokenVersion,
      // iat과 exp는 JwtModule의 설정으로 생성된다.
    });
    return { accessToken, expiresIn: 900 };
  }
}

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(config: ConfigService, private readonly usersService: UsersService) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      secretOrKey: config.getOrThrow<string>('JWT_SECRET'),
      algorithms: ['HS256'],
      ignoreExpiration: false,
      issuer: config.getOrThrow<string>('JWT_ISSUER'),
      audience: config.getOrThrow<string>('JWT_AUDIENCE'),
    });
  }

  async validate(payload: unknown): Promise<AuthenticatedUser> {
    // TypeScript 타입은 토큰의 런타임 스키마를 검증하지 않는다.
    // exp 없는 토큰도 거부한다. ignoreExpiration: false만으로 exp가 필수가 되지는 않는다.
    if (!isAccessTokenClaims(payload)) throw new UnauthorizedException();

    const user = await this.usersService.findAuthUserById(payload.sub);
    if (!user || !user.isActive || user.tokenVersion !== payload.tokenVersion) {
      throw new UnauthorizedException();
    }

    // 현재 역할만 반환하고 비밀번호 해시 등 내부 필드는 request.user에 넣지 않는다.
    return { id: user.id, roles: user.roles };
  }
}

@Module({
  imports: [
    ConfigModule,
    UsersModule,
    PassportModule.register({ defaultStrategy: 'jwt' }),
    JwtModule.registerAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        secret: config.getOrThrow<string>('JWT_SECRET'),
        signOptions: {
          algorithm: 'HS256',
          expiresIn: '15m',
          issuer: config.getOrThrow<string>('JWT_ISSUER'),
          audience: config.getOrThrow<string>('JWT_AUDIENCE'),
        },
      }),
    }),
  ],
  providers: [AuthService, JwtStrategy],
  exports: [AuthService, PassportModule],
})
export class AuthModule {}
```

`JWT_SECRET`에는 암호학적으로 생성한 충분한 엔트로피의 키를 사용하고, 환경 설정 검증에서 빈 값과 누락을 거부한다. `getOrThrow()`만으로 키의 강도를 보장하지는 않는다. 발급자와 검증 서버의 시계를 동기화하고, 허용할 시계 오차가 있다면 서명 검증과 클레임 검사에 같은 정책을 적용한다.

비밀번호 변경·전체 로그아웃·계정 차단에 따른 `tokenVersion` 증가는 관련 상태 변경과 같은 DB 트랜잭션에서 처리한다. 초 단위 `iat` 비교만으로는 같은 초 안에 발급된 토큰을 정확히 무효화할 수 없다. 버전 변경은 이미 실행 중인 요청을 취소하지는 않으며, 사용자 조회에 캐시나 지연 복제본을 쓰면 그만큼 권한 철회가 늦어진다.

리프레시 토큰이 필요하면 다음 동작까지 구현한다. 위의 액세스 토큰 발급 예제는 갱신 동작을 제공하지 않는다.

1. 32바이트 이상의 난수 토큰을 발급하고 원문 대신 조회 가능한 SHA-256 해시를 유일 키로 저장한다. 기록에는 사용자·세션/토큰 계열·만료·사용/철회 상태와 발급 당시 `tokenVersion`을 둔다.
2. 갱신 시 현재 사용자 상태와 버전을 다시 확인한다. 조건부 UPDATE나 행 잠금으로 동시 갱신을 직렬화하고, 기존 토큰의 단회 소비와 새 토큰 저장을 한 트랜잭션으로 처리한다.
3. 이미 소비된 토큰이 다시 들어오면 계열을 철회한다. 철회 트랜잭션을 커밋한 뒤 401을 반환해야 예외로 철회까지 롤백되지 않는다. 로그아웃·비밀번호 변경도 해당 갱신 권한을 철회한다.
4. 쿠키로 전달한다면 `HttpOnly`·`Secure`·적절한 `SameSite`와 CSRF 방어를 함께 적용한다. 토큰 원문은 로그에 남기지 않는다.


참고: [Passport JWT](https://www.passportjs.org/packages/passport-jwt/), [jsonwebtoken 검증 옵션](https://github.com/auth0/node-jsonwebtoken#jwtverifytoken-secretorpublickey-options-callback), [OAuth 보안 권고의 리프레시 토큰 보호](https://www.rfc-editor.org/rfc/rfc9700.html#section-4.14)
