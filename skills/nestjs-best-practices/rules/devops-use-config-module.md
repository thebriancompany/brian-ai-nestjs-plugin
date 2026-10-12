---
title: 환경 설정에 ConfigModule을 사용한다
impact: LOW-MEDIUM
impactDescription: "올바른 설정으로 배포 실패를 방지함"
tags: devops, configuration, environment, validation
---

## 환경 설정에 ConfigModule을 사용한다

환경별 설정에 `@nestjs/config`를 사용한다. 시작 시 설정을 검증해 잘못된 설정을 빠르게 발견한다. 네임스페이스별 설정으로 구성을 정리하고 타입 안전성을 확보한다.

**잘못된 예 (process.env에 직접 접근함):**

```typescript
// process.env에 직접 접근
@Injectable()
export class DatabaseService {
  constructor() {
    // 검증이 없어 런타임에 실패할 수 있음
    this.connection = new Pool({
      host: process.env.DB_HOST,
      port: parseInt(process.env.DB_PORT), // 없으면 NaN
      password: process.env.DB_PASSWORD, // 없으면 undefined
    });
  }
}

// 여러 곳에 흩어진 환경 변수 접근
@Injectable()
export class EmailService {
  sendEmail() {
    // 서비스마다 다른 방식으로 환경 변수에 접근
    const apiKey = process.env.SENDGRID_API_KEY || 'default';
    // 오타를 발견하지 못함: process.env.SENDGRID_API_KY
  }
}
```

**올바른 예 (검증과 함께 @nestjs/config를 사용함):**

```typescript
// 검증을 적용한 설정 구성
import { ConfigModule, ConfigService, registerAs } from '@nestjs/config';
import * as Joi from 'joi';

// config/database.config.ts
export const databaseConfig = registerAs('database', () => ({
  host: process.env.DB_HOST,
  port: parseInt(process.env.DB_PORT, 10),
  username: process.env.DB_USERNAME,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME,
}));

// config/app.config.ts
export const appConfig = registerAs('app', () => ({
  port: parseInt(process.env.PORT, 10) || 3000,
  environment: process.env.NODE_ENV || 'development',
  apiPrefix: process.env.API_PREFIX || 'api',
}));

// config/validation.schema.ts
export const validationSchema = Joi.object({
  NODE_ENV: Joi.string()
    .valid('development', 'production', 'test')
    .default('development'),
  PORT: Joi.number().default(3000),
  DB_HOST: Joi.string().required(),
  DB_PORT: Joi.number().default(5432),
  DB_USERNAME: Joi.string().required(),
  DB_PASSWORD: Joi.string().required(),
  DB_NAME: Joi.string().required(),
  JWT_SECRET: Joi.string().min(32).required(),
  REDIS_URL: Joi.string().uri().required(),
});

// app.module.ts
@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true, // 모듈을 가져오지 않아도 어디서나 사용 가능
      load: [databaseConfig, appConfig],
      validationSchema,
      validationOptions: {
        abortEarly: true, // 첫 오류에서 중단
        allowUnknown: true, // 다른 환경 변수 허용
      },
    }),
    TypeOrmModule.forRootAsync({
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        type: 'postgres',
        host: config.get('database.host'),
        port: config.get('database.port'),
        username: config.get('database.username'),
        password: config.get('database.password'),
        database: config.get('database.database'),
        autoLoadEntities: true,
      }),
    }),
  ],
})
export class AppModule {}

// 타입 안전한 설정 접근
export interface AppConfig {
  port: number;
  environment: 'development' | 'production' | 'test';
  apiPrefix: string;
}

export interface DatabaseConfig {
  host: string;
  port: number;
  username: string;
  password: string;
  database: string;
}

// 타입 안전한 접근
@Injectable()
export class AppService {
  constructor(private config: ConfigService) {}

  getPort(): number {
    // 제네릭으로 타입 안전성 확보
    return this.config.get<number>('app.port');
  }

  getDatabaseConfig(): DatabaseConfig {
    return this.config.get<DatabaseConfig>('database');
  }
}

// 네임스페이스 설정을 직접 주입
@Injectable()
export class DatabaseService {
  constructor(
    @Inject(databaseConfig.KEY)
    private dbConfig: ConfigType<typeof databaseConfig>,
  ) {
    // 완전한 타입 추론!
    const host = this.dbConfig.host; // string
    const port = this.dbConfig.port; // number
  }
}

// 환경 파일 지원
ConfigModule.forRoot({
  envFilePath: [
    `.env.${process.env.NODE_ENV}.local`,
    `.env.${process.env.NODE_ENV}`,
    '.env.local',
    '.env',
  ],
});

// .env.development
// DB_HOST=localhost
// DB_PORT=5432

// .env.production
// DB_HOST=prod-db.example.com
// DB_PORT=5432
```

참고: [NestJS Configuration](https://docs.nestjs.com/techniques/configuration)
