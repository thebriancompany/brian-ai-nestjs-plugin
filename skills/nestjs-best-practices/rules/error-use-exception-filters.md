---
title: 예외 필터로 오류를 처리한다
impact: HIGH
impactDescription: "일관되고 중앙화된 오류 처리를 제공함"
tags: error-handling, exception-filters, consistency
---

## 예외 필터로 오류를 처리한다

컨트롤러에서 예외를 잡아 오류 응답 형식을 직접 만들지 않는다. NestJS 예외 필터로 애플리케이션 전반의 오류를 일관되게 처리한다. 특정 오류 타입을 위한 사용자 정의 예외 필터와 처리되지 않은 예외를 위한 전역 필터를 만든다.

**잘못된 예 (컨트롤러에서 수동으로 오류를 처리함):**

```typescript
// 컨트롤러에서 수동으로 오류 처리
@Controller('users')
export class UsersController {
  @Get(':id')
  async findOne(@Param('id') id: string, @Res() res: Response) {
    try {
      const user = await this.usersService.findById(id);
      if (!user) {
        return res.status(404).json({
          statusCode: 404,
          message: 'User not found',
        });
      }
      return res.json(user);
    } catch (error) {
      console.error(error);
      return res.status(500).json({
        statusCode: 500,
        message: 'Internal server error',
      });
    }
  }
}
```

**올바른 예 (예외 필터로 일관되게 처리함):**

```typescript
// 내장 예외와 사용자 정의 예외 사용
@Controller('users')
export class UsersController {
  @Get(':id')
  async findOne(@Param('id') id: string): Promise<User> {
    const user = await this.usersService.findById(id);
    if (!user) {
      throw new NotFoundException(`User #${id} not found`);
    }
    return user;
  }
}

// 사용자 정의 도메인 예외
export class UserNotFoundException extends NotFoundException {
  constructor(userId: string) {
    super({
      statusCode: 404,
      error: 'Not Found',
      message: `User with ID "${userId}" not found`,
      code: 'USER_NOT_FOUND',
    });
  }
}

// 도메인 오류를 위한 사용자 정의 예외 필터
@Catch(DomainException)
export class DomainExceptionFilter implements ExceptionFilter {
  catch(exception: DomainException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();

    const status = exception.getStatus?.() || 400;

    response.status(status).json({
      statusCode: status,
      code: exception.code,
      message: exception.message,
      timestamp: new Date().toISOString(),
      path: request.url,
    });
  }
}

// 처리되지 않은 오류를 위한 전역 예외 필터
@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  constructor(private readonly logger: Logger) {}

  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();

    const status =
      exception instanceof HttpException
        ? exception.getStatus()
        : HttpStatus.INTERNAL_SERVER_ERROR;

    const message =
      exception instanceof HttpException
        ? exception.message
        : 'Internal server error';

    this.logger.error(
      `${request.method} ${request.url}`,
      exception instanceof Error ? exception.stack : exception,
    );

    response.status(status).json({
      statusCode: status,
      message,
      timestamp: new Date().toISOString(),
      path: request.url,
    });
  }
}

// main.ts에서 전역 등록
app.useGlobalFilters(
  new AllExceptionsFilter(app.get(Logger)),
  new DomainExceptionFilter(),
);

// 또는 모듈을 통해 등록
@Module({
  providers: [
    {
      provide: APP_FILTER,
      useClass: AllExceptionsFilter,
    },
  ],
})
export class AppModule {}
```

참고: [NestJS Exception Filters](https://docs.nestjs.com/exception-filters)
