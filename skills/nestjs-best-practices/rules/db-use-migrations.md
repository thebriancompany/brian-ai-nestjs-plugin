---
title: 데이터베이스 마이그레이션을 사용한다
impact: HIGH
impactDescription: "안전하고 반복 가능한 데이터베이스 스키마 변경을 지원함"
tags: database, migrations, typeorm, schema
---

## 데이터베이스 마이그레이션을 사용한다

운영 환경에서는 절대 `synchronize: true`를 사용하지 않는다. 모든 스키마 변경에 마이그레이션을 사용한다. 마이그레이션으로 변경 이력을 관리하고 환경 간 일관성을 유지한다. 데이터를 삭제한 변경은 down()만으로 복구되지 않으므로 롤백 가능 범위와 백업·전진 수정 절차를 구분한다.

**잘못된 예 (synchronize나 수동 SQL을 사용함):**

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
// 마이그레이션을 위한 TypeORM 설정
// data-source.ts
export const dataSource = new DataSource({
  type: 'postgres',
  host: process.env.DB_HOST,
  port: parseInt(process.env.DB_PORT),
  username: process.env.DB_USERNAME,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME,
  entities: ['dist/**/*.entity.js'],
  migrations: ['dist/migrations/*.js'],
  synchronize: false, // 운영 환경에서는 항상 false
  migrationsRun: false, // 배포 파이프라인의 단일 실행 단계에서 적용
});

// app.module.ts
TypeOrmModule.forRootAsync({
  inject: [ConfigService],
  useFactory: (config: ConfigService) => ({
    type: 'postgres',
    host: config.get('DB_HOST'),
    synchronize: false, // 마이그레이션을 사용하는 환경은 같은 방식으로 관리
    migrations: ['dist/migrations/*.js'],
    migrationsRun: false,
  }),
});

// migrations/1705312800000-AddUserAge.ts
import { MigrationInterface, QueryRunner } from 'typeorm';

export class AddUserAge1705312800000 implements MigrationInterface {
  name = 'AddUserAge1705312800000';

  public async up(queryRunner: QueryRunner): Promise<void> {
    // 기존 행을 처리할 수 있도록 기본값과 함께 컬럼 추가
    await queryRunner.query(`
      ALTER TABLE "users" ADD "age" integer DEFAULT 0
    `);

    // 자주 조회하는 컬럼에 인덱스 추가
    await queryRunner.query(`
      CREATE INDEX "IDX_users_age" ON "users" ("age")
    `);
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    // 롤백을 위해 항상 down 구현
    await queryRunner.query(`DROP INDEX "IDX_users_age"`);
    await queryRunner.query(`ALTER TABLE "users" DROP COLUMN "age"`);
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

대용량 테이블은 backfill을 작은 배치로 나누고 잠금 시간을 측정한다. 마이그레이션 파일은 위 배포 단계별로 순차 추가하며, 세 파일을 처음부터 한 번에 자동 실행하지 않는다.

참고: [TypeORM Migrations](https://typeorm.io/migrations)
