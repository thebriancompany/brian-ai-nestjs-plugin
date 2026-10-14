---
title: 인증과 인가에 가드를 사용한다
impact: HIGH
impactDescription: "핸들러 실행 전에 접근 제어를 적용함"
tags: security, guards, authentication, authorization
---

## 인증과 인가에 가드를 사용한다

가드는 인증 상태, 역할, 권한 등의 조건에 따라 요청을 처리할지 결정한다. 미들웨어 다음, 파이프와 인터셉터 이전에 실행되므로 접근 제어에 적합하다. 컨트롤러에서 수동으로 확인하는 대신 가드를 사용한다.

**잘못된 예 (모든 핸들러에서 인증과 인가를 직접 확인함):**

```typescript
// 모든 핸들러에서 인증과 인가를 수동 확인
@Controller('admin')
export class AdminController {
  @Get('users')
  async getUsers(@Request() req) {
    if (!req.user) {
      throw new UnauthorizedException();
    }
    if (!req.user.roles.includes('admin')) {
      throw new ForbiddenException();
    }
    return this.adminService.getUsers();
  }

  @Delete('users/:id')
  async deleteUser(@Request() req, @Param('id') id: string) {
    if (!req.user) {
      throw new UnauthorizedException();
    }
    if (!req.user.roles.includes('admin')) {
      throw new ForbiddenException();
    }
    return this.adminService.deleteUser(id);
  }
}
```

**올바른 예 (선언적인 데코레이터와 가드를 사용함):**

인증은 [JWT 인증 규칙](security-auth-jwt.md)의 `JwtStrategy`에 위임한다. 해당 전략을 `AuthModule`의 프로바이더로 등록하고 `UsersService`를 주입할 수 있도록 구성한다. 전략은 사용자 존재·활성 상태와 필수 클레임·토큰 버전에 따른 무효화 조건을 확인하고, 현재 사용자 정보를 반환한다. `request.user`에는 이 반환값이 들어가므로 역할 가드도 토큰 발급 당시가 아닌 현재 역할을 검사한다.

```typescript
import { AuthGuard } from '@nestjs/passport';

// JWT 인증 가드
@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  constructor(private reflector: Reflector) {
    super();
  }

  canActivate(context: ExecutionContext) {
    // @Public() 데코레이터 확인
    const isPublic = this.reflector.getAllAndOverride<boolean>('isPublic', [
      context.getHandler(),
      context.getClass(),
    ]);
    if (isPublic) return true;

    // 서명·만료 검증 뒤 JwtStrategy.validate()로 현재 사용자 상태를 확인한다.
    return super.canActivate(context);
  }
}

// 역할 가드
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<Role[]>('roles', [
      context.getHandler(),
      context.getClass(),
    ]);

    const isPublic = this.reflector.getAllAndOverride<boolean>('isPublic', [
      context.getHandler(), context.getClass(),
    ]);
    if (isPublic || !requiredRoles?.length) return true;

    const { user } = context.switchToHttp().getRequest<{ user?: AuthenticatedUser }>();
    if (!user) throw new UnauthorizedException();
    return requiredRoles.some((role) => user.roles.includes(role));
  }
}

// 데코레이터
export const Public = () => SetMetadata('isPublic', true);
export const Roles = (...roles: Role[]) => SetMetadata('roles', roles);

// 가드 전역 등록: 같은 모듈에서 인증 → 역할 순서를 보장한다.
@Module({
  imports: [AuthModule], // PassportModule과 JwtStrategy가 구성된 모듈
  providers: [
    { provide: APP_GUARD, useClass: JwtAuthGuard },
    { provide: APP_GUARD, useClass: RolesGuard },
  ],
})
export class AppModule {}

// 간결한 컨트롤러
@Controller('admin')
@Roles(Role.Admin) // 모든 라우트에 적용
export class AdminController {
  constructor(private readonly adminService: AdminService) {}

  @Get('users')
  getUsers(): Promise<User[]> {
    return this.adminService.getUsers();
  }

  @Delete('users/:id')
  deleteUser(@Param('id', new ParseUUIDPipe({ version: '4' })) id: string): Promise<void> {
    return this.adminService.deleteUser(id);
  }

  @Public() // 재정의: 인증 불필요
  @Get('health')
  health() {
    return { status: 'ok' };
  }
}
```

`@Public()`은 위 두 가드를 모두 건너뛰는 명시적 예외다. `@Roles()`는 지정한 역할 중 하나를 요구하고, 메서드의 역할 목록은 클래스 목록을 대체한다. 역할 검사를 통과해도 특정 레코드의 소유자·테넌트 접근 권한은 별도로 검증해야 한다. 가드는 파이프보다 먼저 실행되므로 가드에서 경로 ID를 사용할 때는 아직 `ParseUUIDPipe`가 적용되지 않았다는 점도 고려한다.

이 예제는 HTTP 전용이다. GraphQL·WebSocket·RPC에는 각각의 실행 컨텍스트와 인증 정보 전달 방식에 맞는 가드를 사용한다. 모든 전역 가드를 하나의 등록 지점에서 조합해 중복 등록을 피한다.

참고: [NestJS Guards](https://docs.nestjs.com/guards)
