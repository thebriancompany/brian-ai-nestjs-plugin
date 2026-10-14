---
title: 공통 관심사에 인터셉터를 사용한다
impact: MEDIUM-HIGH
impactDescription: "핸들러의 응답 스트림을 감싸 공통 동작을 적용함"
tags: api, interceptors, logging, timeout
---

## 공통 관심사에 인터셉터를 사용한다

인터셉터는 가드 이후 핸들러 실행 전후의 공통 동작에 적합하다. 응답 변환·측정·시간 제한을 일관되게 적용하되 비즈니스 처리는 서비스에 둔다. 가드나 미들웨어에서 끝난 요청은 인터셉터가 관찰하지 못하므로 전체 HTTP 접근 로그는 미들웨어나 HTTP 로거에서 처리한다.

**잘못된 예 (모든 핸들러에 같은 측정 코드를 반복함):**

```typescript
@Get()
async findAll() {
  const start = Date.now();
  const users = await this.usersService.findAll();
  this.logger.log(`Finished in ${Date.now() - start}ms`);
  return users; // 실패 경로는 측정되지 않는다.
}
```

**올바른 예 (성공·실패·구독 해제를 관찰함):**

```typescript
import { finalize, tap, timeout, catchError } from 'rxjs/operators';
import { TimeoutError, Observable } from 'rxjs';

@Injectable()
export class TimingInterceptor implements NestInterceptor {
  private readonly logger = new Logger(TimingInterceptor.name);

  intercept(context: ExecutionContext, next: CallHandler): Observable<unknown> {
    const request = context.switchToHttp().getRequest();
    const started = performance.now();
    let outcome = 'cancelled';
    return next.handle().pipe(
      tap({
        complete: () => { outcome = 'completed'; },
        error: () => { outcome = 'failed'; },
      }),
      finalize(() => {
        this.logger.log({
          event: 'handler.finished',
          method: request.method,
          route: request.route?.path, // 전체 URL·쿼리·body는 기록하지 않는다.
          outcome,
          durationMs: Math.round(performance.now() - started),
        });
      }),
    );
  }
}

@Module({
  providers: [{ provide: APP_INTERCEPTOR, useClass: TimingInterceptor }],
})
export class AppModule {}
```

이 시간은 응답 스트림 처리 시간이며 실제 소켓 전송 완료까지의 시간이 아니다. 동일 인터셉터를 전역과 컨트롤러에 중복 등록하지 않는다. DI가 필요하면 `APP_INTERCEPTOR` 또는 클래스 기반 `@UseInterceptors`로 등록한다.

단일 응답 JSON API에는 다음처럼 응답 대기 시간을 제한할 수 있다.

```typescript
@Injectable()
export class TimeoutInterceptor implements NestInterceptor {
  intercept(_context: ExecutionContext, next: CallHandler): Observable<unknown> {
    return next.handle().pipe(
      timeout({ first: 5000 }),
      catchError((error: unknown) => {
        if (error instanceof TimeoutError) {
          throw new GatewayTimeoutException('Operation timed out');
        }
        throw error;
      }),
    );
  }
}

@Controller('reports')
@UseInterceptors(TimeoutInterceptor)
export class ReportsController {
  @Get(':id')
  findOne(@Param('id', ParseUUIDPipe) id: string) {
    return this.reportsService.findOne(id);
  }
}
```

RxJS 시간 초과는 구독을 해제하지만 이미 실행 중인 Promise·DB 쿼리·결제를 자동 취소하지 않는다. 드라이버의 시간 제한과 취소 신호를 함께 사용하고, 변경 요청의 재시도에는 멱등성을 적용한다. `first`는 첫 응답까지의 제한이므로 무한 스트림 전체 수명을 제한하는 설정으로 사용하지 않는다.

응답을 `{ data, meta }`로 감쌀 때는 해당 계약을 가진 JSON 라우트에만 적용한다. 파일 다운로드, SSE, 204, 직접 `@Res()`를 사용하는 라우트에 전역 변환을 적용하면 응답이 깨질 수 있다. 인터셉터는 요청에서 등록 순서로, 응답에서 역순으로 실행된다. DTO를 먼저 직렬화하고 감싸려면 응답 포장 인터셉터를 직렬화 인터셉터보다 바깥에 둔다. [응답 DTO](api-use-dto-serialization.md), [캐싱](perf-use-caching.md), [예외 필터](error-use-exception-filters.md)의 구현을 재사용하고 유사한 처리를 여러 계층에 중복하지 않는다.

참고: [NestJS 11 Interceptors](https://docs.nestjs.com/v11/interceptors), [RxJS timeout](https://rxjs.dev/api/operators/timeout)
