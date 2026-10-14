---
title: HTTP 경계에 맞는 예외를 던진다
impact: HIGH
impactDescription: "실패를 명확히 전달하고 도메인 오류와 서버 장애를 구분함"
tags: error-handling, exceptions, services
---

## HTTP 경계에 맞는 예외를 던진다

HTTP 전용 서비스는 `HttpException`의 하위 클래스를 던져 실패 상태를 전달할 수 있다. 여러 전송 방식에서 재사용하는 도메인 서비스에는 도메인 예외를 사용하고 HTTP 경계에서 매핑한다. 프로젝트에서 타입 안전한 Result 패턴을 쓰는 것도 유효하지만, 성공과 실패를 구분하기 어려운 임의의 선택적 필드 조합은 피한다.

**잘못된 예 (실패 여부가 모호한 반환형):**

```typescript
@Injectable()
export class UsersService {
  async findById(id: string): Promise<{ user?: User; error?: string }> {
    const user = await this.repo.findOne({ where: { id } });
    if (!user) return { error: 'User not found' };
    return { user };
  }
}

@Controller('users')
export class UsersController {
  @Get(':id')
  async findOne(@Param('id') id: string) {
    const result = await this.usersService.findById(id);
    // 호출자가 오류 확인을 잊으면 성공 응답으로 처리될 수 있다.
    return result.user;
  }
}
```

**올바른 예 (예상한 오류만 적절한 HTTP 예외로 변환함):**

아래는 PostgreSQL과 TypeORM을 사용하는 사용자 관리 예제다. `CreateUserDto`에는 검증된 `email`, `name`만 있고, DB 마이그레이션에서 `users.email`에 이름이 `UQ_users_email`인 유일 제약을 만든다. 비밀번호 등록·해시는 별도의 인증 흐름에서 처리한다.

```typescript
import { QueryFailedError, Repository } from 'typeorm';
import { InjectRepository } from '@nestjs/typeorm';

@Injectable()
export class UsersService {
  constructor(@InjectRepository(User) private readonly repo: Repository<User>) {}

  async findById(id: string): Promise<User> {
    const user = await this.repo.findOne({ where: { id } });
    if (!user) throw new NotFoundException('User not found');
    return user;
  }

  async create(dto: CreateUserDto): Promise<User> {
    try {
      return await this.repo.save(this.repo.create({
        email: dto.email,
        name: dto.name,
      }));
    } catch (error: unknown) {
      // 사전 SELECT만으로는 동시 생성의 중복을 막지 못한다.
      // 이 드라이버의 알려진 제약 위반만 409로 매핑한다.
      if (error instanceof QueryFailedError) {
        const cause = error.driverError as { code?: string; constraint?: string };
        if (cause.code === '23505' && cause.constraint === 'UQ_users_email') {
          throw new ConflictException('Email already registered');
        }
      }
      throw error; // 연결 장애나 다른 SQL 오류를 409로 숨기지 않는다.
    }
  }
}

@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Get(':id')
  findOne(@Param('id', new ParseUUIDPipe({ version: '4' })) id: string): Promise<User> {
    return this.usersService.findById(id);
  }

  @Post()
  create(@Body() dto: CreateUserDto): Promise<User> {
    return this.usersService.create(dto);
  }
}
```

드라이버의 오류 코드와 제약 이름은 실제 DB에 맞춰 확인한다. 이 예제의 저장과 충돌 처리는 독립된 작업이며, 이미 실패한 PostgreSQL 트랜잭션 안에서 후속 쿼리를 계속 실행해서는 안 된다. 이메일 정규화 규칙도 검증·저장·유일 제약 사이에서 일치해야 한다. 공개 가입·로그인 API는 계정 존재 여부 노출 정책을 별도로 정한다.

인가 가드, DTO 검증, 민감한 필드를 제외하는 응답 직렬화는 별도로 구성한다. 도메인 예외가 필요하면 [예외 필터 규칙](error-use-exception-filters.md)처럼 구체적인 오류만 매핑한다.

참고: [NestJS Exception Filters](https://docs.nestjs.com/exception-filters), [PostgreSQL 오류 코드](https://www.postgresql.org/docs/current/errcodes-appendix.html)
