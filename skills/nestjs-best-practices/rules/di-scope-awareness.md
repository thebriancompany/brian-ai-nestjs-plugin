---
title: 프로바이더 스코프를 이해한다
impact: CRITICAL
impactDescription: "데이터 유출과 성능 문제를 방지함"
tags: dependency-injection, scopes, request-context
---

## 프로바이더 스코프를 이해한다

NestJS에는 DEFAULT(기본 싱글턴), REQUEST(요청 문맥별 인스턴스), TRANSIENT(소비자별 전용 인스턴스) 스코프가 있다. 상태가 없는 서비스와 공유 연결 풀은 기본 스코프로 두는 경우가 많다. 요청 스코프는 이를 주입받는 소비자에게 전파되지만 트랜지언트 스코프는 같은 방식으로 전파되지 않는다. 싱글턴이 주입받은 트랜지언트 인스턴스는 그 싱글턴에 유지되므로 요청마다 새로 생기거나 메서드 호출마다 초기화되지 않는다.

**잘못된 예 (잘못된 스코프 사용):**

```typescript
// 불필요한 요청 스코프 사용으로 성능 저하
@Injectable({ scope: Scope.REQUEST })
export class UsersService {
  // 매 요청마다 새 인스턴스를 생성함
  // 이 서비스를 주입받는 소비자로 요청 스코프가 전파됨
  async findAll() {
    return this.userRepo.find();
  }
}

// 변경 가능한 요청 상태를 가진 싱글턴
@Injectable() // 기본값: 싱글턴
export class RequestContextService {
  private userId: string; // 위험: 모든 요청이 공유함!

  setUser(userId: string) {
    this.userId = userId; // 동시에 처리되는 모든 요청의 값을 덮어씀
  }

  getUser() {
    return this.userId; // 잘못된 사용자를 반환함!
  }
}
```

**올바른 예 (사용 사례에 맞는 스코프 사용):**

```typescript
// 상태가 없는 서비스에는 싱글턴 사용 (기본값이며 가장 흔함)
@Injectable()
export class UsersService {
  constructor(private readonly userRepo: UserRepository) {}

  async findById(id: string): Promise<User | null> {
    return this.userRepo.findOne({ where: { id } });
  }
}

// 요청 문맥이 필요한 경우에만 요청 스코프 사용
@Injectable({ scope: Scope.REQUEST })
export class RequestContextService {
  private userId: string | undefined;

  setUser(userId: string) {
    this.userId = userId;
  }

  getUser(): string | undefined {
    return this.userId;
  }
}

// 요청 객체가 필요한 경우: NestJS 내장 요청 문맥 사용
import { REQUEST } from '@nestjs/core';
import type { Request } from 'express';

type AuthenticatedRequest = Request & { user?: { id: string } };

@Injectable({ scope: Scope.REQUEST })
export class AuditService {
  constructor(@Inject(REQUEST) private readonly request: AuthenticatedRequest) {}

  log(action: string) {
    console.log(`User ${this.request.user?.id} performed ${action}`);
  }
}
```

인증된 사용자 정보는 인증 가드가 `request.user`를 설정한 뒤 읽는다. 생성자나 인증 전 미들웨어에서 값을 복사해 두면 요청 스코프여도 사용자가 아직 없을 수 있다. 사용자 입력 헤더를 그대로 인증된 사용자나 테넌트 ID로 취급하지 않는다.

**대안 (요청별 문맥을 초기화한 CLS 사용):**

```typescript
import { ClsService } from 'nestjs-cls';

@Injectable() // 싱글턴을 유지함!
export class ClsAuditService {
  constructor(private readonly cls: ClsService) {}

  log(action: string) {
    const userId = this.cls.get('userId');
    console.log(`User ${userId} performed ${action}`);
  }
}
```

CLS는 요청별 store를 초기화하고 인증 이후 `userId`를 채우는 설정이 필요하다. [로깅 규칙](devops-use-logging.md)의 미들웨어·인터셉터 순서를 함께 적용한다. 작업 큐나 예약 작업에는 HTTP 요청 문맥이 자동으로 존재하지 않으므로 별도 문맥을 만들거나 필요한 값을 명시적으로 전달한다. Passport 전략 등 싱글턴을 전제로 하는 프로바이더에 요청 스코프 의존성을 바로 주입하지 않는다.

참고: [NestJS Injection Scopes](https://docs.nestjs.com/fundamentals/injection-scopes)
