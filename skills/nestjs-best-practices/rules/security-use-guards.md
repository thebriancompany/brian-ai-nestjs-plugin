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

```typescript
// JWT 인증 가드
@Injectable()
export class JwtAuthGuard implements CanActivate {
  constructor(
    private jwtService: JwtService,
    private reflector: Reflector,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    // @Public() 데코레이터 확인
    const isPublic = this.reflector.getAllAndOverride<boolean>('isPublic', [
      context.getHandler(),
      context.getClass(),
    ]);
    if (isPublic) return true;

    const request = context.switchToHttp().getRequest();
    const token = this.extractToken(request);

    if (!token) {
      throw new UnauthorizedException('No token provided');
    }

    try {
      request.user = await this.jwtService.verifyAsync(token);
      return true;
    } catch {
      throw new UnauthorizedException('Invalid token');
    }
  }

  private extractToken(request: Request): string | undefined {
    const [type, token] = request.headers.authorization?.split(' ') ?? [];
    return type === 'Bearer' ? token : undefined;
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

    const { user } = context.switchToHttp().getRequest();
    if (!user) throw new UnauthorizedException();
    return requiredRoles.some((role) => user.roles?.includes(role));
  }
}

// 데코레이터
export const Public = () => SetMetadata('isPublic', true);
export const Roles = (...roles: Role[]) => SetMetadata('roles', roles);

// 가드 전역 등록
@Module({
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
  @Get('users')
  getUsers(): Promise<User[]> {
    return this.adminService.getUsers();
  }

  @Delete('users/:id')
  deleteUser(@Param('id') id: string): Promise<void> {
    return this.adminService.deleteUser(id);
  }

  @Public() // 재정의: 인증 불필요
  @Get('health')
  health() {
    return { status: 'ok' };
  }
}
```

참고: [NestJS Guards](https://docs.nestjs.com/guards)
