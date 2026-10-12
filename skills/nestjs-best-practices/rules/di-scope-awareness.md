---
title: 프로바이더 스코프를 이해한다
impact: CRITICAL
impactDescription: "데이터 유출과 성능 문제를 방지함"
tags: dependency-injection, scopes, request-context
---

## 프로바이더 스코프를 이해한다

NestJS에는 DEFAULT(싱글턴), REQUEST(요청별 인스턴스), TRANSIENT(주입할 때마다 새 인스턴스)라는 세 가지 프로바이더 스코프가 있다. 대부분의 프로바이더는 싱글턴으로 두는 것이 좋다. 요청 스코프는 의존성 트리의 상위로 전파되므로 성능에 영향을 준다. 스코프를 이해하면 메모리 누수와 잘못된 데이터 공유를 방지할 수 있다.

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

  async findById(id: string): Promise<User> {
    return this.userRepo.findOne({ where: { id } });
  }
}

// 요청 문맥이 필요한 경우에만 요청 스코프 사용
@Injectable({ scope: Scope.REQUEST })
export class RequestContextService {
  private userId: string;

  setUser(userId: string) {
    this.userId = userId;
  }

  getUser(): string {
    return this.userId;
  }
}

// 요청 객체가 필요한 경우: NestJS 내장 요청 문맥 사용
import { REQUEST } from '@nestjs/core';
import { Request } from 'express';

@Injectable({ scope: Scope.REQUEST })
export class AuditService {
  constructor(@Inject(REQUEST) private request: Request) {}

  log(action: string) {
    console.log(`User ${this.request.user?.id} performed ${action}`);
  }
}

// 대안: 요청별 컨텍스트 초기화가 설정된 ClsModule 사용 (스코프 전파 없음)
import { ClsService } from 'nestjs-cls';

@Injectable() // 싱글턴을 유지함!
export class AuditService {
  constructor(private cls: ClsService) {}

  log(action: string) {
    const userId = this.cls.get('userId');
    console.log(`User ${userId} performed ${action}`);
  }
}
```

참고: [NestJS Injection Scopes](https://docs.nestjs.com/fundamentals/injection-scopes)
