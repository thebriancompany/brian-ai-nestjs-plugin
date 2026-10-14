---
title: 호환성을 깨뜨리는 변경에는 API 버전을 사용한다
impact: MEDIUM
impactDescription: "기존 클라이언트 계약을 유지하면서 API를 변경함"
tags: api, versioning, breaking-changes, compatibility
---

## 호환성을 깨뜨리는 변경에는 API 버전을 사용한다

필드 제거·이름 변경처럼 기존 클라이언트 계약을 깨뜨리는 변경은 기존 버전을 유지한 채 새 버전에 적용한다. 선택 필드 추가 등 호환 가능한 변경마다 버전을 늘릴 필요는 없다. URI, 헤더, 미디어 타입 중 한 전략을 선택하고 실제 요청으로 라우팅을 검증한다.

**잘못된 예 (같은 경로의 응답 계약을 교체함):**

```typescript
// GET /users/:id
// 이전: { id, name, email }
// 변경: { id, firstName, lastName, emailAddress }
// 기존 클라이언트가 읽는 필드가 사라진다.
```

**올바른 예 (버전별 핸들러에서 응답을 명시적으로 매핑함):**

```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.enableVersioning({ type: VersioningType.URI, defaultVersion: '1' });
  await app.listen(3000);
}

@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Get(':id')
  @Version('1')
  async findOneV1(@Param('id', ParseUUIDPipe) id: string) {
    const user = await this.usersService.findById(id);
    if (!user) throw new NotFoundException('User not found');
    return { id: user.id, name: user.name, email: user.email };
  }

  @Get(':id')
  @Version('2')
  async findOneV2(@Param('id', ParseUUIDPipe) id: string) {
    const user = await this.usersService.findById(id);
    if (!user) throw new NotFoundException('User not found');
    return {
      id: user.id,
      firstName: user.firstName,
      lastName: user.lastName,
      emailAddress: user.email,
    };
  }
}

@Controller('health')
export class HealthController {
  @Get('live')
  @Version(VERSION_NEUTRAL)
  @Public()
  liveness() { return { status: 'ok' }; }
}
```

URI 전략에서 위 경로는 `/v1/users/:id`, `/v2/users/:id`, `/health/live`다. `VERSION_NEUTRAL`은 URI에 모든 버전 접두사를 붙여 준다는 뜻이 아니다. `defaultVersion: '1'`도 `/users/:id`를 자동 별칭으로 만들지 않는다. 이미 버전 없는 API를 운영 중이라면 그 경로를 별도 중립 라우트나 게이트웨이 매핑으로 유지해야 한다. 전역 접두사가 있다면 버전 앞에 붙는다.

헤더 전략은 `enableVersioning({ type: VersioningType.HEADER, header: 'X-API-Version' })`로 대신 설정하며 URI 전략과 혼용하지 않는다. 필요한 버전 헤더가 없거나 지원하지 않는 버전이면 404가 되는지 테스트한다. 여러 버전의 계약이 실제로 같을 때만 `@Version(['1', '2'])`로 핸들러를 공유한다. 수동 `/v1` 경로도 가능하지만 내장 버전 관리와 중복 적용하지 않는다.

지원 중단은 클라이언트 사용량을 확인하고 공지한 일정에 따라 진행한다. 응답 헤더를 사용하면 `Deprecation`은 RFC 9745의 날짜 형식(`@` 뒤 Unix 초), `Sunset`은 HTTP-date를 사용하고 실제 종료 일정과 맞춘다. 지원 중단 예고만으로 구버전을 즉시 제거하지 않는다.

참고: [NestJS 11 Versioning](https://docs.nestjs.com/v11/techniques/versioning), [Deprecation header](https://www.rfc-editor.org/rfc/rfc9745.html)
