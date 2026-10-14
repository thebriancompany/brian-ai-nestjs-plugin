---
title: 환경 설정에 ConfigModule을 사용한다
impact: MEDIUM-HIGH
impactDescription: "잘못된 설정을 시작 시 거부해 배포 후 장애를 줄임"
tags: devops, configuration, environment, validation
---

## 환경 설정에 ConfigModule을 사용한다

설정 읽기와 검증을 시작 경계에 모은다. `process.env`를 읽는 것 자체가 문제는 아니다. 서비스마다 변환과 기본값 처리를 반복하거나 필수 비밀 값에 임의의 기본값을 넣으면 잘못된 설정으로 앱이 시작될 수 있다. `ConfigService.get<T>()`의 제네릭은 런타임 검증이나 형 변환을 수행하지 않는다.

**잘못된 예 (검증 없이 변환하고 비밀 값에 기본값을 사용함):**

```typescript
const port = Number(process.env.DB_PORT); // 누락되면 NaN
const apiKey = process.env.SENDGRID_API_KEY || 'default';
```

**올바른 예 (설정 팩터리에서 검증한 값을 반환함):**

네임스페이스 설정 팩터리는 반환값을 직접 검증한다. 루트 `validationSchema`의 변환·기본값을 별도로 읽는 `process.env`에 적용됐다고 가정하지 않는다.

```typescript
import { ConfigModule, ConfigType, registerAs } from '@nestjs/config';
import * as Joi from 'joi';

interface DatabaseConfig {
  host: string;
  port: number;
  username: string;
  password: string;
  database: string;
}

export const databaseConfig = registerAs('database', (): DatabaseConfig => {
  const schema = Joi.object({
    host: Joi.string().required(),
    port: Joi.number().integer().min(1).max(65535).default(5432),
    username: Joi.string().required(),
    password: Joi.string().required(),
    database: Joi.string().required(),
  });
  const { value, error } = schema.validate({
    host: process.env.DB_HOST,
    port: process.env.DB_PORT,
    username: process.env.DB_USERNAME,
    password: process.env.DB_PASSWORD,
    database: process.env.DB_NAME,
  }, { abortEarly: false, convert: true });
  if (error) {
    // 원래 환경 변수나 비밀번호 값을 오류 로그에 넣지 않는다.
    const fields = error.details.map(detail => detail.path.join('.')).join(', ');
    throw new Error(`Invalid database configuration fields: ${fields}`);
  }
  return value as DatabaseConfig; // 위 스키마로 필드·타입·기본값을 검증한 결과
});

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      // 배포 플랫폼이 환경 변수를 제공하는 운영 환경에서는 .env를 읽지 않는다.
      ignoreEnvFile: process.env.NODE_ENV === 'production',
      load: [databaseConfig],
    }),
    TypeOrmModule.forRootAsync({
      inject: [databaseConfig.KEY],
      useFactory: (db: ConfigType<typeof databaseConfig>) => ({
        type: 'postgres',
        ...db,
        autoLoadEntities: true,
        synchronize: false, // 스키마 변경은 검토한 마이그레이션으로 적용
      }),
    }),
  ],
})
export class AppModule {}

@Injectable()
export class DatabaseSettings {
  constructor(
    @Inject(databaseConfig.KEY)
    private readonly db: ConfigType<typeof databaseConfig>,
  ) {}

  getPort(): number { return this.db.port; }
}
```

HTTP 포트, 토큰 issuer·audience, 외부 서비스 URL도 사용하는 모듈에서 타입과 허용 범위를 검증한다. 필요한 키만 검증하고 사용하지 않는 Redis 등의 설정을 필수로 만들지 않는다. 문자열 키로 조회해야 한다면 검증된 설정에 `getOrThrow()`를 사용해 누락을 거부한다. 비밀 값은 배포 환경의 비밀 저장소에서 주입하고 저장소에 커밋하지 않는다. `.env` 파일 이름만으로 운영·테스트 환경이 격리되지는 않는다.

참고: [NestJS Configuration](https://docs.nestjs.com/techniques/configuration)
