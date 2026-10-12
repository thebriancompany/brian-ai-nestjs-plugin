---
title: 호환성을 깨뜨리는 변경에는 API 버전을 사용한다
impact: MEDIUM
impactDescription: "기존 클라이언트를 깨뜨리지 않고 API를 발전시킬 수 있게 함"
tags: api, versioning, breaking-changes, compatibility
---

## 호환성을 깨뜨리는 변경에는 API 버전을 사용한다

API에 호환성을 깨뜨리는 변경을 적용할 때 NestJS의 내장 버전 관리 기능을 사용한다. URI, 헤더, 미디어 타입 중 버전 관리 전략을 선택해 일관되게 적용한다. 이렇게 하면 새 클라이언트가 변경된 엔드포인트를 사용하는 동안 기존 클라이언트도 계속 동작할 수 있다.

**잘못된 예 (버전 관리 없이 호환성을 깨뜨림):**

```typescript
// 버전 관리 없이 호환성을 깨뜨리는 변경
@Controller('users')
export class UsersController {
  @Get(':id')
  async findOne(@Param('id') id: string): Promise<User> {
    // 기존 응답: { id, name, email }
    // 이후 변경된 응답: { id, firstName, lastName, emailAddress }
    // 기존 클라이언트가 깨짐!
    return this.usersService.findOne(id);
  }
}

// 라우트에서 수동으로 버전 관리
@Controller('v1/users')
export class UsersV1Controller {}

@Controller('v2/users')
export class UsersV2Controller {}
// 일관성이 없고 오류가 나기 쉬우며 유지보수하기 어려움
```

**올바른 예 (NestJS 내장 버전 관리 기능을 사용함):**

```typescript
// main.ts에서 버전 관리 활성화
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // URI 버전 관리: /v1/users, /v2/users
  app.enableVersioning({
    type: VersioningType.URI,
    defaultVersion: '1',
  });

  // HEADER나 MEDIA_TYPE은 URI의 대안이다. enableVersioning은 한 전략으로 설정한다.
  // 예: { type: VersioningType.HEADER, header: 'X-API-Version', defaultVersion: '1' }

  await app.listen(3000);
}

// 버전별 컨트롤러
@Controller({ path: 'users', version: '1' })
export class UsersV1Controller {
  @Get(':id')
  async findOne(@Param('id') id: string): Promise<UserV1Response> {
    const user = await this.usersService.findOne(id);
    // V1 응답 형식
    return {
      id: user.id,
      name: user.name,
      email: user.email,
    };
  }
}

@Controller({ path: 'users', version: '2' })
export class UsersV2Controller {
  @Get(':id')
  async findOne(@Param('id') id: string): Promise<UserV2Response> {
    const user = await this.usersService.findOne(id);
    // 호환성을 깨뜨리는 V2 응답 형식
    return {
      id: user.id,
      firstName: user.firstName,
      lastName: user.lastName,
      emailAddress: user.email,
      createdAt: user.createdAt,
    };
  }
}

// 라우트별 버전 관리: 라우트마다 다른 버전 사용
@Controller('users')
export class UsersController {
  @Get()
  @Version('1')
  findAllV1(): Promise<UserV1Response[]> {
    return this.usersService.findAllV1();
  }

  @Get()
  @Version('2')
  findAllV2(): Promise<UserV2Response[]> {
    return this.usersService.findAllV2();
  }

  @Get(':id')
  @Version(['1', '2']) // 여러 버전에서 같은 핸들러 사용
  findOne(@Param('id') id: string): Promise<User> {
    return this.usersService.findOne(id);
  }

  @Post()
  @Version(VERSION_NEUTRAL) // 모든 버전에서 사용 가능
  create(@Body() dto: CreateUserDto): Promise<User> {
    return this.usersService.create(dto);
  }
}

// 버전별 로직을 담은 공유 서비스
@Injectable()
export class UsersService {
  async findOneV1(id: string): Promise<UserV1Response> {
    const user = await this.repo.findOne({ where: { id } });
    return this.toV1Response(user);
  }

  async findOneV2(id: string): Promise<UserV2Response> {
    const user = await this.repo.findOne({ where: { id } });
    return this.toV2Response(user);
  }

  private toV1Response(user: User): UserV1Response {
    return {
      id: user.id,
      name: `${user.firstName} ${user.lastName}`,
      email: user.email,
    };
  }

  private toV2Response(user: User): UserV2Response {
    return {
      id: user.id,
      firstName: user.firstName,
      lastName: user.lastName,
      emailAddress: user.email,
      createdAt: user.createdAt,
    };
  }
}

// 위 컨트롤러 예제의 대안: Nest가 선택한 버전별 핸들러에서 서비스 호출
// URI 전략에서는 /v1/users/:id, /v2/users/:id로 선택하며 헤더는 사용하지 않는다.
// HEADER 전략에서는 X-API-Version으로 선택하며 미지원 버전은 404로 거부한다.
@Controller('users')
export class UsersController {
  @Get(':id')
  @Version('1')
  findOneV1(@Param('id') id: string): Promise<UserV1Response> {
    return this.usersService.findOneV1(id);
  }

  @Get(':id')
  @Version('2')
  findOneV2(@Param('id') id: string): Promise<UserV2Response> {
    return this.usersService.findOneV2(id);
  }
}

// 지원 중단 전략: 이전 버전을 사용 중단 예정으로 표시
@Controller({ path: 'users', version: '1' })
@UseInterceptors(DeprecationInterceptor)
export class UsersV1Controller {
  // 모든 V1 라우트에 지원 중단 경고 포함
}

@Injectable()
export class DeprecationInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const response = context.switchToHttp().getResponse();
    response.setHeader('Deprecation', 'true');
    response.setHeader('Sunset', 'Sat, 1 Jan 2025 00:00:00 GMT');
    response.setHeader('Link', '</v2/users>; rel="successor-version"');

    return next.handle();
  }
}
```

참고: [NestJS Versioning](https://docs.nestjs.com/techniques/versioning)
