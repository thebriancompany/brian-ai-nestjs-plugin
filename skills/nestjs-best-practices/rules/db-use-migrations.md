---
title: 데이터베이스 마이그레이션을 사용한다
impact: HIGH
impactDescription: "안전하고 반복 가능한 데이터베이스 스키마 변경을 지원함"
tags: database, migrations, typeorm, schema
---

## 데이터베이스 마이그레이션을 사용한다

운영 환경에서는 절대 `synchronize: true`를 사용하지 않는다. 모든 스키마 변경에 마이그레이션을 사용한다. 마이그레이션으로 변경 이력을 관리하고 환경 간 일관성을 유지한다. 데이터를 삭제한 변경은 down()만으로 복구되지 않으므로 롤백 가능 범위와 백업·전진 수정 절차를 구분한다.

**잘못된 예 (운영 동기화나 이력이 없는 스키마 변경을 사용함):**

```typescript
// 운영 환경에서 synchronize 사용
TypeOrmModule.forRoot({
  type: 'postgres',
  synchronize: true, // 운영 환경에서 위험함!
  // 컬럼, 테이블, 데이터가 삭제될 수 있음
});

// 운영 환경의 수동 SQL
@Injectable()
export class DatabaseService {
  async addColumn(): Promise<void> {
    await this.dataSource.query('ALTER TABLE users ADD COLUMN age INT');
    // 버전 관리와 롤백이 없고 환경마다 일관되지 않음
  }
}

// 마이그레이션 없이 엔티티 수정
@Entity()
export class User {
  @Column()
  email: string;

  @Column() // 마이그레이션 없이 추가됨
  newField: string; // synchronize가 false이면 운영 환경에서 오류 발생
}
```

**올바른 예 (모든 스키마 변경에 마이그레이션을 사용함):**

```typescript
// database-options.ts: CLI와 애플리케이션이 같은 DB·엔티티·마이그레이션을 사용한다.
// CommonJS 빌드 예제. ESM에서는 import.meta.url에 맞게 경로를 계산한다.
import { join } from 'node:path';
import { DataSourceOptions } from 'typeorm';
import { User } from './users/user.entity';

export function databaseOptions(url: string): DataSourceOptions {
  if (!url) throw new Error('DATABASE_URL is required');
  return {
    type: 'postgres',
    url,
    entities: [User], // 프로젝트의 모든 엔티티를 명시한다.
    migrations: [join(__dirname, 'migrations', '*.{ts,js}')],
    synchronize: false,
    migrationsRun: false, // 배포 파이프라인의 단일 실행 단계에서 적용한다.
  };
}

// data-source.ts (별도 파일): CLI는 Nest의 ConfigModule을 실행하지 않는다.
// DATABASE_URL은 배포 환경 또는 명시적인 환경 파일 로더로 제공한다.
import { DataSource } from 'typeorm';
import { databaseOptions } from './database-options';

export default new DataSource(databaseOptions(process.env.DATABASE_URL ?? ''));

// app.module.ts: ConfigModule.forRoot()에서 환경을 읽고 검증한 뒤 사용한다.
TypeOrmModule.forRootAsync({
  imports: [ConfigModule],
  inject: [ConfigService],
  useFactory: (config: ConfigService) =>
    databaseOptions(config.getOrThrow<string>('DATABASE_URL')),
});

// migrations/1705312800000-AddUserAge.ts
import { MigrationInterface, QueryRunner } from 'typeorm';

export class AddUserAge1705312800000 implements MigrationInterface {
  name = 'AddUserAge1705312800000';

  public async up(queryRunner: QueryRunner): Promise<void> {
    // 기존 사용자의 미확인 나이를 임의의 0으로 바꾸지 않는다.
    await queryRunner.query('ALTER TABLE "users" ADD "age" integer');
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    // 컬럼에 쓰인 데이터는 삭제된다. 실제 롤백 전 보존 필요성을 확인한다.
    await queryRunner.query('ALTER TABLE "users" DROP COLUMN "age"');
  }
}

// 배포 1: 새 컬럼만 추가한다. 구버전의 name 읽기/쓰기를 계속 허용한다.
// 이 마이그레이션만 먼저 배포한다. 아래 수축 단계와 같은 릴리스에 넣지 않는다.
export class AddFullName1705312900000 implements MigrationInterface {
  async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query('ALTER TABLE "users" ADD "full_name" varchar(255)');
  }
  async down(queryRunner: QueryRunner): Promise<void> {
    // 새 컬럼에 고유한 데이터가 생긴 뒤에는 데이터 이관 없이 실행하지 않는다.
    await queryRunner.query('ALTER TABLE "users" DROP COLUMN "full_name"');
  }
}

// 배포 2: 구버전 writer가 존재하는 동안은 name을 읽고 두 컬럼에 같은 값을 원자적으로 쓴다.
// 구버전 writer가 모두 사라진 후 최종 backfill을 수행한다.
// 구버전이 name만 수정할 수 있는 동안은 먼저 채운 full_name이 오래된 값일 수 있다.
// 따라서 아래 backfill은 구버전 writer 제거 후에 수행한다.
export class BackfillFullName1705313000000 implements MigrationInterface {
  async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query('UPDATE "users" SET "full_name" = "name"');
    await queryRunner.query('ALTER TABLE "users" ALTER COLUMN "full_name" SET NOT NULL');
    // 모든 reader가 full_name으로 전환된 다음 새 writer의 단일 컬럼 쓰기를 허용한다.
    await queryRunner.query('ALTER TABLE "users" ALTER COLUMN "name" DROP NOT NULL');
  }
  async down(): Promise<void> {
    throw new Error('Forward repair required: reconcile both columns before rollback');
  }
}

// 배포 3: full_name만 사용하는 코드로 전환하고, 이전 코드/백그라운드 작업이
// name을 더 이상 사용하지 않는지 확인한다. 롤백 기간이 끝난 별도 릴리스에서만 실행한다.
export class DropLegacyName1705313100000 implements MigrationInterface {
  async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query('ALTER TABLE "users" DROP COLUMN "name"');
  }
  async down(): Promise<void> {
    throw new Error('Destructive migration: restore from backup or apply forward repair');
  }
}
```

개발 환경에서는 스키마 차이를 생성한 뒤 SQL을 검토한다. 실행 대상은 변경 전 스키마여야 하며, 생성 명령을 실행했다는 사실만으로 새/기존 환경 모두에 적용 가능한지는 증명되지 않는다.

```bash
# TypeORM 0.3, CommonJS/ts-node 프로젝트의 개발 단계
npx typeorm-ts-node-commonjs migration:generate src/migrations/AddUserAge -d src/data-source.ts
# 빌드가 끝난 배포 작업에서 한 번 실행 (dist 경로는 실제 빌드 산출물에 맞춘다)
npx typeorm migration:run -d dist/data-source.js
```

빌드 전에 이전 dist를 정리해 삭제/이동된 마이그레이션 JS가 남지 않게 한다. TypeScript 소스와 컴파일 결과를 같은 디렉터리에 두지 않는다. 이미 적용된 마이그레이션을 수정하지 말고 새 파일로 전진 수정한다. 성공하기 전에는 새 스키마가 필요한 앱 버전을 준비 상태로 전환하지 않는다.

대용량 테이블은 backfill을 작은 배치로 나누고 잠금 시간을 측정한다. 마이그레이션 파일은 위 배포 단계별로 순차 추가하며, 세 파일을 처음부터 한 번에 자동 실행하지 않는다.

검증: 빈 DB와 직전 운영 스키마 양쪽에서 migration:run을 실행하고, 구버전/신버전 코드가 공존하는 단계의 읽기·쓰기를 확인한다. down이 데이터 복구를 보장하지 않는 변경은 백업 복구 또는 전진 수정 절차를 별도로 시험한다.

참고: [TypeORM Migrations](https://typeorm.io/docs/advanced-topics/migrations/)
