---
title: 서비스에서 HTTP 예외를 던진다
impact: HIGH
impactDescription: "컨트롤러를 간결하게 유지하고 오류 처리를 단순화함"
tags: error-handling, exceptions, services
---

## 서비스에서 HTTP 예외를 던진다

HTTP 애플리케이션에서는 서비스에서 `HttpException`의 하위 클래스를 던져도 되며, 이 방식이 더 나은 경우도 많다. 컨트롤러를 간결하게 유지하고 서비스가 적절한 오류 상태를 전달할 수 있다. 특정 계층에 의존하지 않아야 하는 서비스에서는 HTTP 상태 코드에 매핑할 도메인 예외를 사용한다.

**잘못된 예 (예외를 던지는 대신 오류 객체를 반환함):**

```typescript
// 예외를 던지는 대신 오류 객체 반환
@Injectable()
export class UsersService {
  async findById(id: string): Promise<{ user?: User; error?: string }> {
    const user = await this.repo.findOne({ where: { id } });
    if (!user) {
      return { error: 'User not found' }; // 컨트롤러에서 확인해야 함
    }
    return { user };
  }
}

@Controller('users')
export class UsersController {
  @Get(':id')
  async findOne(@Param('id') id: string) {
    const result = await this.usersService.findById(id);
    if (result.error) {
      throw new NotFoundException(result.error);
    }
    return result.user;
  }
}
```

**올바른 예 (서비스에서 직접 예외를 던짐):**

```typescript
// 서비스에서 직접 예외 발생
@Injectable()
export class UsersService {
  constructor(private readonly repo: UserRepository) {}

  async findById(id: string): Promise<User> {
    const user = await this.repo.findOne({ where: { id } });
    if (!user) {
      throw new NotFoundException(`User #${id} not found`);
    }
    return user;
  }

  async create(dto: CreateUserDto): Promise<User> {
    const existing = await this.repo.findOne({
      where: { email: dto.email },
    });
    if (existing) {
      throw new ConflictException('Email already registered');
    }
    return this.repo.save(dto);
  }

  async update(id: string, dto: UpdateUserDto): Promise<User> {
    const user = await this.findById(id); // 없으면 예외 발생
    Object.assign(user, dto);
    return this.repo.save(user);
  }
}

// 컨트롤러를 간결하게 유지
@Controller('users')
export class UsersController {
  @Get(':id')
  findOne(@Param('id') id: string): Promise<User> {
    return this.usersService.findById(id);
  }

  @Post()
  create(@Body() dto: CreateUserDto): Promise<User> {
    return this.usersService.create(dto);
  }
}

// 계층에 의존하지 않는 서비스에는 도메인 예외 사용
export class EntityNotFoundException extends Error {
  constructor(
    public readonly entity: string,
    public readonly id: string,
  ) {
    super(`${entity} with ID "${id}" not found`);
  }
}

// 예외 필터에서 HTTP 응답으로 매핑
@Catch(EntityNotFoundException)
export class EntityNotFoundFilter implements ExceptionFilter {
  catch(exception: EntityNotFoundException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();

    response.status(404).json({
      statusCode: 404,
      message: exception.message,
      entity: exception.entity,
      id: exception.id,
    });
  }
}
```

참고: [NestJS Exception Filters](https://docs.nestjs.com/exception-filters)
