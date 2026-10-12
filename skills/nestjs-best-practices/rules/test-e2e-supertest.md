---
title: E2E 테스트에 Supertest를 사용한다
impact: HIGH
impactDescription: "전체 요청과 응답 주기를 검증함"
tags: testing, e2e, supertest, integration
---

## E2E 테스트에 Supertest를 사용한다

종단 간(E2E) 테스트는 Supertest로 NestJS 애플리케이션에 실제 HTTP 요청을 보낸다. 미들웨어, 가드, 파이프, 인터셉터를 포함한 전체 스택을 테스트한다. E2E 테스트는 단위 테스트에서 놓치는 통합 문제를 찾아낸다.

**잘못된 예 (E2E 테스트의 준비나 정리가 올바르지 않음):**

```typescript
// 컨트롤러 단위 테스트만 수행
describe('UsersController', () => {
  it('should return users', async () => {
    const service = { findAll: jest.fn().mockResolvedValue([]) };
    const controller = new UsersController(service as any);

    const result = await controller.findAll();

    expect(result).toEqual([]);
    // 라우트, 가드, 파이프, 직렬화는 테스트하지 않음
  });
});

// 올바른 준비와 정리가 없는 E2E 테스트
describe('Users API', () => {
  it('should create user', async () => {
    const app = await NestFactory.create(AppModule);
    // 올바른 초기화 없음
    // 테스트 후 정리 없음
    // 실제 데이터베이스에 접근
  });
});
```

**올바른 예 (Supertest를 사용해 E2E 테스트를 올바르게 준비함):**

```typescript
// 올바른 E2E 테스트 설정
import { Test, TestingModule } from '@nestjs/testing';
import { INestApplication, ValidationPipe } from '@nestjs/common';
import * as request from 'supertest';
import { AppModule } from '../src/app.module';

describe('UsersController (e2e)', () => {
  let app: INestApplication;

  beforeAll(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleFixture.createNestApplication();

    // 운영 환경과 같은 설정 적용
    app.useGlobalPipes(
      new ValidationPipe({
        whitelist: true,
        transform: true,
        forbidNonWhitelisted: true,
      }),
    );

    await app.init();
  });

  afterAll(async () => {
    await app.close();
  });

  describe('/users (POST)', () => {
    it('should create a user', () => {
      return request(app.getHttpServer())
        .post('/users')
        .send({ name: 'John', email: 'john@test.com' })
        .expect(201)
        .expect((res) => {
          expect(res.body).toHaveProperty('id');
          expect(res.body.name).toBe('John');
          expect(res.body.email).toBe('john@test.com');
        });
    });

    it('should return 400 for invalid email', () => {
      return request(app.getHttpServer())
        .post('/users')
        .send({ name: 'John', email: 'invalid-email' })
        .expect(400)
        .expect((res) => {
          expect(res.body.message).toContain('email');
        });
    });
  });

  describe('/users/:id (GET)', () => {
    it('should return 404 for non-existent user', () => {
      return request(app.getHttpServer())
        .get('/users/non-existent-id')
        .expect(404);
    });
  });
});

// 인증을 포함한 테스트
describe('Protected Routes (e2e)', () => {
  let app: INestApplication;
  let authToken: string;

  beforeAll(async () => {
    const moduleFixture = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleFixture.createNestApplication();
    app.useGlobalPipes(new ValidationPipe({ whitelist: true }));
    await app.init();

    // 인증 토큰 발급
    const loginResponse = await request(app.getHttpServer())
      .post('/auth/login')
      .send({ email: 'test@test.com', password: 'password' });

    authToken = loginResponse.body.accessToken;
  });

  it('should return 401 without token', () => {
    return request(app.getHttpServer())
      .get('/users/me')
      .expect(401);
  });

  it('should return user profile with valid token', () => {
    return request(app.getHttpServer())
      .get('/users/me')
      .set('Authorization', `Bearer ${authToken}`)
      .expect(200)
      .expect((res) => {
        expect(res.body.email).toBe('test@test.com');
      });
  });
});

// E2E 테스트의 데이터베이스 격리
// CI가 테스트 작업마다 독립적인 임시 DB를 생성하고 TEST_DATABASE_URL로 제공한다.
// 운영 DB와 같은 자격 증명을 사용하지 않는다. .env.test 파일 이름만으로는 격리가 보장되지 않는다.
describe('Orders API (e2e)', () => {
  let app: INestApplication;

  beforeAll(async () => {
    if (process.env.NODE_ENV !== 'test' || !process.env.TEST_DATABASE_URL) {
      throw new Error('An isolated test database is required');
    }
    // TestAppModule은 TEST_DATABASE_URL만 사용하고 synchronize:false로 설정한다.
    const moduleFixture = await Test.createTestingModule({ imports: [TestAppModule] }).compile();
    app = moduleFixture.createNestApplication();
    await moduleFixture.get(DataSource).runMigrations();
    await app.init();
  });

  // 테스트마다 고유한 데이터를 만들고, 해당 테스트가 만든 데이터만 정리한다.
  // 테스트 작업이 끝나면 CI가 임시 DB를 폐기한다.
  afterAll(async () => { await app?.close(); });
});
```

참고: [NestJS E2E Testing](https://docs.nestjs.com/fundamentals/testing#end-to-end-testing)
