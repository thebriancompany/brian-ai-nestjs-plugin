---
title: 안전한 종료를 구현한다
impact: MEDIUM-HIGH
impactDescription: "올바른 종료 처리가 무중단 배포를 가능하게 함"
tags: devops, graceful-shutdown, lifecycle, kubernetes
---

## 안전한 종료를 구현한다

SIGTERM과 SIGINT 신호를 처리해 NestJS 애플리케이션을 안전하게 종료한다. 새 요청 수신을 중단하고, 처리 중인 요청이 완료될 때까지 기다린 뒤, 데이터베이스 연결을 닫고 자원을 정리한다. 배포 중 데이터 손실과 연결 오류를 방지할 수 있다.

**잘못된 예 (종료 신호를 무시함):**

```typescript
// 종료 신호 무시
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
  // SIGTERM을 받으면 앱이 즉시 종료됨
  // 처리 중인 요청이 실패함
  // 데이터베이스 연결이 갑자기 끊김
}

// 취소할 수 없는 장기 실행 작업
@Injectable()
export class ProcessingService {
  async processLargeFile(file: File): Promise<void> {
    // 종료 중 이 작업을 중단할 방법이 없음
    for (let i = 0; i < file.chunks.length; i++) {
      await this.processChunk(file.chunks[i]);
      // 수 분간 실행되며 종료를 막을 수 있음
    }
  }
}
```

**올바른 예 (HTTP 요청을 비운 뒤 Nest 생명주기 훅을 실행함):**

다음 예제는 Express HTTP 앱에서 종료를 한 곳에서 조율한다. `app.close()`는 모듈 종료 훅부터 실행하므로, 요청이 사용하는 DB를 닫는 훅보다 먼저 요청을 비워야 한다. 이 방식을 사용할 때 `enableShutdownHooks()`를 함께 켜면 신호 처리 경로가 중복된다.

```typescript
import { finalize } from 'rxjs/operators';
import type { Request, Response, NextFunction } from 'express';

@Injectable()
export class ShutdownService {
  private shuttingDown = false;
  startShutdown(): void { this.shuttingDown = true; }
  isShutdown(): boolean { return this.shuttingDown; }
}

@Injectable()
export class RequestTracker implements NestMiddleware, NestInterceptor {
  private responses = 0;
  private handlers = 0;
  private beforeHandlers = 0;
  private starts = new WeakMap<object, () => void>();
  private waiters: Array<() => void> = [];

  constructor(private readonly shutdown: ShutdownService) {}

  private isProbe(req: Request): boolean {
    return req.path === '/health/live' || req.path === '/health/ready';
  }

  use(req: Request, res: Response, next: NextFunction): void {
    if (this.isProbe(req)) { next(); return; }
    if (this.shutdown.isShutdown()) {
      res.status(503).send('Service Unavailable');
      return;
    }
    this.responses++;
    this.beforeHandlers++;
    let started = false;
    const releaseBeforeHandler = () => {
      if (started) return;
      started = true;
      this.beforeHandlers--;
      this.starts.delete(req);
      this.notifyDrained();
    };
    this.starts.set(req, releaseBeforeHandler);
    // 가드에서 거부된 요청도 정상 응답 완료 시 추적에서 해제한다.
    res.once('finish', releaseBeforeHandler);
    let released = false;
    const release = () => {
      if (released) return;
      released = true;
      res.off('finish', release);
      res.off('close', release);
      this.responses--;
      this.notifyDrained();
    };
    res.once('finish', release);
    res.once('close', release);
    next();
  }

  intercept(context: ExecutionContext, next: CallHandler): Observable<unknown> {
    if (this.isProbe(context.switchToHttp().getRequest())) return next.handle();
    // 클라이언트가 연결을 끊어도 실행 중인 핸들러를 따로 기다린다.
    this.handlers++;
    this.starts.get(context.switchToHttp().getRequest())?.();
    try {
      return next.handle().pipe(finalize(() => {
        this.handlers--;
        this.notifyDrained();
      }));
    } catch (error) {
      this.handlers--;
      this.notifyDrained();
      throw error;
    }
  }

  drain(): Promise<void> {
    if (this.responses === 0 && this.handlers === 0 && this.beforeHandlers === 0) return Promise.resolve();
    return new Promise(resolve => this.waiters.push(resolve));
  }

  private notifyDrained(): void {
    if (this.responses === 0 && this.handlers === 0 && this.beforeHandlers === 0) {
      for (const resolve of this.waiters.splice(0)) resolve();
    }
  }
}

@Injectable()
export class ShutdownCoordinator {
  private pending?: Promise<void>;
  constructor(
    private readonly state: ShutdownService,
    private readonly tracker: RequestTracker,
  ) {}

  shutdown(app: INestApplication, propagationMs = 5000): Promise<void> {
    if (this.pending) return this.pending;
    this.state.startShutdown(); // 다음 readiness 검사부터 503
    this.pending = (async () => {
      // 로드 밸런서의 엔드포인트 갱신 시간. 배포 환경에 맞춰 조정한다.
      await new Promise(resolve => setTimeout(resolve, propagationMs));
      await this.tracker.drain(); // DB 등 요청에 필요한 자원은 아직 열려 있다.
      await app.close(); // Nest의 자원 정리를 정확히 한 번 실행한다.
    })();
    return this.pending;
  }
}

@Module({
  providers: [ShutdownService, RequestTracker, ShutdownCoordinator],
  exports: [ShutdownService, RequestTracker, ShutdownCoordinator],
})
export class ShutdownModule {}

// AppModule과 상태 검사 기능 모듈에서 ShutdownModule을 가져온다.
// 기능 모듈에서 ShutdownService를 다시 providers에 등록하지 않는다.
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  const tracker = app.get(RequestTracker);
  app.use(tracker.use.bind(tracker));
  app.useGlobalInterceptors(tracker);
  const coordinator = app.get(ShutdownCoordinator);
  await app.listen(3000); // 시작이 끝난 뒤 종료 핸들러를 설치한다.
  let requested = false;
  for (const signal of ['SIGTERM', 'SIGINT']) {
    process.on(signal, () => {
      if (requested) return;
      requested = true;
      const deadline = setTimeout(() => process.exit(1), 30000);
      deadline.unref();
      void coordinator.shutdown(app)
        .then(() => clearTimeout(deadline))
        .catch(error => {
          console.error(error);
          process.exitCode = 1;
          // 종료 훅 실패로 활성 핸들이 남아도 제한 시간에 강제 종료한다.
          // 실패 경로에서는 deadline을 해제하지 않는다.
        });
    });
  }
}
```

시작 중에는 기본 신호 종료 동작을 유지한다. 비동기 초기화가 끝나기 전에 `app.close()`를 실행하면 뒤늦은 `listen()`으로 서버가 다시 열릴 수 있으므로 종료 핸들러를 시작 완료 후 등록한다.

프로브 경로는 실제 전역 접두사와 맞춘다. 준비 상태 엔드포인트는 `ShutdownService.isShutdown()`이면 503을 반환한다([상태 검사 규칙](micro-use-health-checks.md)). 추적기와 상태 객체는 앱 전체에서 같은 인스턴스를 사용한다. 요청이 시작한 필수 비동기 작업은 반드시 반환하거나 `await`해야 한다. 반환하지 않은 작업과 별도 스트림의 수명은 이 추적기가 알 수 없다. 시간 초과 인터셉터처럼 구독을 먼저 해제하는 코드가 있으면 `finalize` 이후에도 원래 Promise가 실행될 수 있다. 그런 작업은 서비스에서 별도로 등록·추적하고 종료 시 실제 완료 또는 취소 완료를 기다려야 한다. 이 추적기만으로 취소되지 않은 결제·DB 작업까지 안전하게 비웠다고 판단하지 않는다. 가드·미들웨어 실행 중 연결이 끊기고 핸들러에도 진입하지 않은 요청은 완료 여부를 알 수 없으므로 강제 종료 제한까지 보수적으로 기다린다.

이 예제의 범위는 HTTP다. 큐 워커와 WebSocket을 함께 운영한다면 신규 작업을 멈추고 워커의 진행 중인 작업도 비운 다음 `app.close()`를 호출하도록 조율한다. BullMQ의 생산자 `Queue.close()`만으로 워커 작업 완료를 기다릴 수는 없다. 무한 스트림과 장기 작업은 취소·재시도 정책이 필요하다. 강제 종료 제한은 전파 대기 시간과 작업 최대 시간을 포함하고, Kubernetes의 `terminationGracePeriodSeconds`는 이 제한보다 길게 설정한다.

참고: [NestJS Lifecycle Events](https://docs.nestjs.com/fundamentals/lifecycle-events)
