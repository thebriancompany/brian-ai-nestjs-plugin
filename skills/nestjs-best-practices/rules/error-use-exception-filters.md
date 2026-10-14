---
title: 예외 필터로 오류를 처리한다
impact: HIGH
impactDescription: "프레임워크의 오류 처리를 보존하면서 도메인 오류를 매핑함"
tags: error-handling, exception-filters, consistency
---

## 예외 필터로 오류를 처리한다

일반적인 HTTP 오류는 NestJS의 기본 예외 처리로 충분하다. `HttpException`의 응답 본문과 ValidationPipe의 오류 배열을 보존하고, 알려지지 않은 오류는 내부 정보를 노출하지 않는 500 응답으로 처리한다. 도메인 예외를 HTTP로 매핑하거나 별도의 응답 계약이 필요한 경우에만 필터를 추가한다.

**잘못된 예 (모든 오류를 같은 응답으로 바꿈):**

```typescript
@Controller('users')
export class UsersController {
  @Get(':id')
  async findOne(@Param('id') id: string, @Res() res: Response) {
    try {
      return res.json(await this.usersService.findById(id));
    } catch (error) {
      // 서비스가 던진 NotFoundException과 ConflictException까지 500으로 바뀐다.
      return res.status(500).json({ message: 'Internal server error' });
    }
  }
}

// 광범위한 필터에서 exception.message만 사용하면 DTO 검증 오류 배열도 사라진다.
const message = exception instanceof HttpException
  ? exception.message
  : 'Internal server error';
```

**올바른 예 (알려진 도메인 예외만 HTTP로 매핑함):**

```typescript
import { ArgumentsHost, Catch, Module, NotFoundException } from '@nestjs/common';
import { APP_FILTER, BaseExceptionFilter, HttpAdapterHost } from '@nestjs/core';

// HTTP에 의존하지 않는 서비스에서 사용하는 도메인 예외
export class UserNotFoundError extends Error {
  constructor() {
    super('User not found');
  }
}

@Catch(UserNotFoundError)
export class UserNotFoundFilter extends BaseExceptionFilter {
  constructor(adapterHost: HttpAdapterHost) {
    super(adapterHost.httpAdapter);
  }

  catch(exception: UserNotFoundError, host: ArgumentsHost): void {
    // BaseExceptionFilter가 현재 HTTP 어댑터로 응답하고 이미 보낸 헤더도 처리한다.
    super.catch(
      new NotFoundException({
        statusCode: 404,
        message: 'User not found',
        code: 'USER_NOT_FOUND',
      }),
      host,
    );
  }
}

@Module({
  providers: [{ provide: APP_FILTER, useClass: UserNotFoundFilter }],
})
export class AppModule {}
```

HTTP에 종속되어도 되는 서비스는 `NotFoundException`을 직접 던지면 되므로 이 매핑 계층이 필수는 아니다. 다른 예외는 기본 필터가 처리하므로 검증 오류의 상세 메시지와 `HttpException.getResponse()`의 필드가 유지되고, 알 수 없는 오류는 기본 500 응답과 서버 로그로 처리된다. `APP_FILTER` 등록은 DI를 사용할 수 있으며, 같은 필터를 `useGlobalFilters()`에도 중복 등록하지 않는다.

예외 필터는 여러 필터가 차례로 예외를 가공하는 파이프라인이 아니다. 한 필터가 처리하면 다른 필터로 다시 전달되지 않는다. 별도의 catch-all 필터가 필요하면 `getResponse()` 본문·HTTP 어댑터의 오류·이미 전송된 응답을 보존하고, URL 쿼리·토큰·본문을 무조건 로그에 남기지 않는다. 오류 메시지에는 내부 DB 정보나 자격 증명을 넣지 않는다.

위 예제는 HTTP용이다. WebSocket과 RPC에는 해당 전송 방식의 필터를 사용한다.

참고: [NestJS Exception Filters](https://docs.nestjs.com/exception-filters), [NestJS 11 기본 필터 구현](https://github.com/nestjs/nest/blob/v11.1.6/packages/core/exceptions/base-exception-filter.ts)
