---
title: E2E 테스트에 Supertest를 사용한다
impact: HIGH
impactDescription: "실제 요청 파이프라인과 인증·검증·응답 계약을 확인함"
tags: testing, e2e, supertest, integration
---

## E2E 테스트에 Supertest를 사용한다

컨트롤러를 직접 호출하는 단위 테스트는 유효하지만 라우팅·가드·파이프·직렬화까지 검증하지는 않는다. Supertest로 초기화한 Nest 앱에 요청하고, 운영과 같은 앱 설정을 공유한다. `main.ts`에서만 등록한 파이프는 `createNestApplication()`으로 자동 적용되지 않는다.

**잘못된 예 (앱과 데이터 격리를 준비하지 않음):**

```typescript
it('creates a user', async () => {
  const app = await NestFactory.create(AppModule);
  // 실제 DB·메일·큐 연결 가능. app.init(), 인증 준비, app.close()도 누락됨.
});
```

**올바른 예 (설정·데이터·수명을 명시한 E2E 테스트):**

다음 예제는 `/auth/login`이 200과 `{ accessToken }`을 반환하고 `/users/me`가 인증을 요구하는 앱의 계약을 검증한다. 프로젝트 계약에 맞게 경로·DTO·기대 상태를 조정한다.

```typescript
// src/configure-app.ts: main.ts와 E2E에서 같은 함수를 호출한다.
export function configureApp(app: INestApplication): void {
  app.useGlobalPipes(new ValidationPipe({
    whitelist: true,
    forbidNonWhitelisted: true,
    transform: true,
    transformOptions: { enableImplicitConversion: false },
  }));
  // 실제 앱의 전역 접두사, 버전, 직렬화 설정도 여기에 둔다.
  // listen()과 종료 신호 등록은 main.ts에서만 수행한다.
}
```

```typescript
import { randomUUID } from 'node:crypto';
import { Test } from '@nestjs/testing';
import request from 'supertest'; // esModuleInterop: true 기준
import { configureApp } from '../src/configure-app';
import { TestAppModule } from './fixtures/test-app.module';
import { seedUser } from './fixtures/seed-user';

// CI가 이 실행 전용 DB를 생성해 TEST_DATABASE_URL로 전달한다.
// TestAppModule은 앱의 기능 모듈·가드를 재사용하며 DB URL과 외부 어댑터만 교체한다.
// seedUser는 실제 비밀번호 해시와 활성 계정을 테스트 DB에 저장하는 헬퍼다.
describe('Authentication (e2e)', () => {
  let app: INestApplication | undefined;
  let accessToken: string;
  const email = `user-${randomUUID()}@example.test`;
  const password = 'test-only-valid-password';

  beforeAll(async () => {
    if (process.env.NODE_ENV !== 'test' || !process.env.TEST_DATABASE_URL) {
      throw new Error('An isolated test database is required');
    }
    const moduleRef = await Test.createTestingModule({
      imports: [TestAppModule],
    }).compile();
    app = moduleRef.createNestApplication();
    configureApp(app);
    await app.get(DataSource).runMigrations();
    await seedUser(app.get(DataSource), { email, password, isActive: true });
    await app.init();

    const login = await request(app.getHttpServer())
      .post('/auth/login')
      .send({ email, password })
      .expect(200);
    expect(login.body.accessToken).toEqual(expect.any(String));
    accessToken = login.body.accessToken;
  });

  afterAll(async () => { await app?.close(); });

  it('rejects requests without credentials', async () => {
    await request(app!.getHttpServer()).get('/users/me').expect(401);
  });

  it('returns the authenticated profile without secret fields', async () => {
    const response = await request(app!.getHttpServer())
      .get('/users/me')
      .set('Authorization', `Bearer ${accessToken}`)
      .expect(200);
    expect(response.body.email).toBe(email);
    expect(response.body).not.toHaveProperty('passwordHash');
    expect(response.body).not.toHaveProperty('tokenVersion');
  });

  it('rejects malformed input before the login handler', async () => {
    const response = await request(app!.getHttpServer())
      .post('/auth/login')
      .send({ email: 'invalid-email', password })
      .expect(400);
    // ValidationPipe의 message는 보통 문자열 배열이다.
    expect(response.body.message).toEqual(
      expect.arrayContaining([expect.stringContaining('email')]),
    );
  });
});
```

테스트 DB에는 운영 자격 증명을 제공하지 않는다. `NODE_ENV` 검사나 `.env.test` 파일 이름만으로 격리가 증명되지는 않는다. CI가 별도 DB/사용자를 생성하고 앱·워커 종료 후 폐기해야 한다. 메일·결제 같은 실제 부수 효과는 테스트 어댑터로 바꾸고 그 교체 범위를 명시한다. 앱 초기화·로그인 실패가 다른 테스트의 401로 가려지지 않도록 준비 단계도 검증한다.

404 검증에는 문법상 유효하지만 저장되지 않은 ID를 사용한다. 잘못된 UUID는 파이프에서 400이 된다. 데이터 변경 테스트는 고유 픽스처를 사용하고 실행 순서에 의존하지 않는다. 인증 규칙에는 만료·잘못된 서명·비활성 사용자·권한 변경·토큰 무효화, 응답 규칙에는 중첩 민감 필드 제외를 포함한다. 테스트 전용 설정 때문에 실제 가드나 직렬화가 빠지지 않도록 확인한다.

참고: [NestJS 11 E2E Testing](https://docs.nestjs.com/v11/fundamentals/testing#end-to-end-testing)
