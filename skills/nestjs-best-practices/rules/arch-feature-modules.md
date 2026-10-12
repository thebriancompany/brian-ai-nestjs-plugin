---
title: 기능별 모듈로 구성한다
impact: CRITICAL
impactDescription: "기능의 변경 범위와 모듈 경계를 파악하기 쉽게 함"
tags: architecture, modules, organization
---

## 기능별 모듈로 구성한다

관련 기능을 캡슐화하는 기능별 모듈로 애플리케이션을 구성한다. 각 기능 모듈은 자체 컨트롤러, 서비스, 엔티티, DTO를 갖추고 독립적으로 구성되어야 한다. 모든 컨트롤러나 모든 서비스를 한곳에 모으는 식의 기술 계층별 구성을 피한다. 작은 앱에서는 단순한 구성으로 시작하고, 서로 다른 변경 이유와 소유권이 생길 때 경계를 나눈다.

**잘못된 예 (기술 계층별 구성):**

```typescript
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

```typescript
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
    ConfigModule.forRoot(),
    TypeOrmModule.forRoot(),
    UsersModule,
    OrdersModule,
    SharedModule,
  ],
})
export class AppModule {}
```

참고: [NestJS Modules](https://docs.nestjs.com/modules)
