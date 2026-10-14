---
title: 기능별 모듈로 구성한다
impact: CRITICAL
impactDescription: "기능의 변경 범위와 모듈 경계를 파악하기 쉽게 함"
tags: architecture, modules, organization
---

## 기능별 모듈로 구성한다

관련 기능을 캡슐화하는 기능별 모듈로 애플리케이션을 구성한다. 컨트롤러, 서비스, 엔티티, DTO는 필요한 기능 모듈에 함께 두되 모든 모듈에 이 계층들을 만들 필요는 없다. 모든 컨트롤러나 모든 서비스를 한곳에 모아 변경 범위가 넓어지는 구성을 피한다. 작은 앱에서는 단순한 구성으로 시작하고, 서로 다른 변경 이유와 소유권이 생길 때 경계를 나눈다.

**잘못된 예 (기술 계층별 구성):**

```text
// 기술 계층별 구성 (안티패턴)
src/
├── controllers/
│   ├── users.controller.ts
│   ├── orders.controller.ts
│   └── products.controller.ts
├── services/
│   ├── users.service.ts
│   ├── orders.service.ts
│   └── products.service.ts
├── entities/
│   ├── user.entity.ts
│   ├── order.entity.ts
│   └── product.entity.ts
└── app.module.ts  // 모든 것을 직접 가져옴
```

**올바른 예 (기능별 모듈 구성):**

```text
// 기능별 모듈 구성
src/
├── users/
│   ├── dto/
│   │   ├── create-user.dto.ts
│   │   └── update-user.dto.ts
│   ├── entities/
│   │   └── user.entity.ts
│   ├── users.controller.ts
│   ├── users.service.ts
│   ├── users.repository.ts
│   └── users.module.ts
├── orders/
│   ├── dto/
│   ├── entities/
│   ├── orders.controller.ts
│   ├── orders.service.ts
│   └── orders.module.ts
├── shared/
│   ├── guards/
│   ├── interceptors/
│   ├── filters/
│   └── shared.module.ts
└── app.module.ts
```

```typescript
// users.module.ts
@Module({
  imports: [TypeOrmModule.forFeature([User])],
  controllers: [UsersController],
  providers: [UsersService, UsersRepository],
  exports: [UsersService], // 다른 곳에 필요한 항목만 내보냄
})
export class UsersModule {}

// app.module.ts
@Module({
  imports: [
    // DB 연결·설정 검증은 루트 구성에서 한 번 등록한다.
    // 이 발췌에서는 해당 구성을 생략한다.
    UsersModule,
    OrdersModule,
    SharedModule,
  ],
})
export class AppModule {}
```

`AppModule`이 `UsersModule`과 `OrdersModule`을 함께 가져오는 것만으로 두 기능 간 주입이 허용되지는 않는다. `OrdersService`가 내보낸 `UsersService`를 사용한다면 `OrdersModule.imports`에도 `UsersModule`을 추가한다. 내부 리포지토리나 엔티티까지 관성적으로 내보내지 않는다.

참고: [NestJS Modules](https://docs.nestjs.com/modules)
