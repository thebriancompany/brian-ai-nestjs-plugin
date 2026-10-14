---
title: 인터페이스에 주입 토큰을 사용한다
impact: HIGH
impactDescription: "런타임에 인터페이스 기반 의존성 주입을 가능하게 함"
tags: dependency-injection, tokens, interfaces
---

## 인터페이스에 주입 토큰을 사용한다

TypeScript 인터페이스는 컴파일 시 사라지므로 런타임 주입 토큰으로 사용할 수 없다. 인터페이스 계약이 필요하면 문자열·심벌 토큰 또는 추상 클래스를 사용하고 해당 토큰의 프로바이더를 등록한다. 구체 클래스를 직접 주입하는 것도 유효하다. 교체 경계가 없는 모든 서비스에 인터페이스를 추가하지 않는다.

**잘못된 예 (인터페이스만으로 주입하려 함):**

```typescript
interface UserReader {
  findById(id: string): Promise<User | null>;
}

@Injectable()
export class UsersService {
  // UserReader는 런타임 토큰이 아니므로 Nest가 의존성을 해석하지 못한다.
  constructor(private readonly users: UserReader) {}
}
```

**올바른 예 (하나의 심벌을 정의하고 등록·주입·내보내기에 재사용함):**

```typescript
// user-reader.ts — 소비자와 모듈이 이 파일의 같은 토큰을 가져온다.
export const USER_READER = Symbol('USER_READER');

export interface UserReader {
  findById(id: string): Promise<User | null>;
}

// database-user-reader.ts — User는 기존 TypeORM 엔티티다.
@Injectable()
export class DatabaseUserReader implements UserReader {
  constructor(@InjectRepository(User) private readonly repo: Repository<User>) {}

  findById(id: string): Promise<User | null> {
    return this.repo.findOneBy({ id });
  }
}

// users.service.ts
@Injectable()
export class UsersService {
  constructor(@Inject(USER_READER) private readonly users: UserReader) {}

  findById(id: string): Promise<User | null> {
    return this.users.findById(id);
  }
}

// users.module.ts
@Module({
  imports: [TypeOrmModule.forFeature([User])],
  providers: [
    { provide: USER_READER, useClass: DatabaseUserReader },
    UsersService,
  ],
  exports: [USER_READER, UsersService],
})
export class UsersModule {}
```

`Symbol('USER_READER')`를 소비자 파일에서 다시 만들면 설명이 같아도 다른 토큰이다. 토큰은 데코레이터가 평가되기 전에 정의되어 있어야 하며 토큰 파일이 서비스나 모듈을 다시 참조하지 않게 한다. `UserReader`는 `import type`으로 가져와도 되지만 `USER_READER`는 값 import여야 한다. 테스트에서는 운영 모듈을 환경 변수로 모의 구현에 전환하기보다 `overrideProvider(USER_READER).useValue(fakeReader)`로 교체한다.

**대안 (추상 클래스를 런타임 토큰으로 사용함):**

아래는 위 심벌 방식과 별개의 구성이다. `UserReaderPort`도 프로바이더 등록이 필요하다.

```typescript
export abstract class UserReaderPort {
  abstract findById(id: string): Promise<User | null>;
}

@Injectable()
export class UsersService {
  constructor(@Inject(UserReaderPort) private readonly users: UserReaderPort) {}

  findById(id: string): Promise<User | null> {
    return this.users.findById(id);
  }
}

@Module({
  imports: [TypeOrmModule.forFeature([User])],
  providers: [
    { provide: UserReaderPort, useClass: DatabaseUserReader },
    UsersService,
  ],
  exports: [UserReaderPort, UsersService],
})
export class UsersModule {}
```

추상 클래스는 값 import로 가져온다. 클래스 메타데이터가 생성되는 구성에서는 `@Inject(UserReaderPort)`를 생략할 수도 있다. 이미 등록된 기본 스코프 `DatabaseUserReader`와 토큰이 같은 인스턴스를 가리켜야 한다면 `providers: [DatabaseUserReader, { provide: USER_READER, useExisting: DatabaseUserReader }]`를 사용한다. 두 토큰을 각각 `useClass`로 등록하면 별도 인스턴스가 생긴다.

참고: [NestJS Custom Providers](https://docs.nestjs.com/fundamentals/custom-providers)
