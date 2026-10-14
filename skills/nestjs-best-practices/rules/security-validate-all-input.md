---
title: DTO와 파이프로 모든 입력을 검증한다
impact: HIGH
impactDescription: "공격에 대한 첫 번째 방어선임"
tags: security, validation, dto, pipes
---

## DTO와 파이프로 모든 입력을 검증한다

프로젝트의 검증 도구로 외부 입력의 타입·범위·크기를 검증한다. class-validator를 쓰는 프로젝트에서는 클래스 DTO와 전역 ValidationPipe를 함께 사용한다. 사용자 입력을 신뢰하지 않는다. 처리 전에 모든 요청 본문, 쿼리 매개변수, 경로 매개변수를 검증한다.

**잘못된 예 (검증 없이 원시 입력을 신뢰함):**

```typescript
// 검증 없이 원시 입력 신뢰
@Controller('users')
export class UsersController {
  @Post()
  create(@Body() body: any) {
    // 예상하지 않은 필드(예: isAdmin)까지 서비스로 전달될 수 있음
    return this.usersService.create(body);
  }

  @Get()
  findAll(@Query() query: any) {
    // query.limit은 배열, 빈 문자열, 음수 또는 과도하게 큰 값일 수 있음
    return this.usersService.findAll(query.limit);
  }
}

// 검증 데코레이터가 없는 DTO
export class CreateUserDto {
  name: string;    // 검증 없음
  email: string;   // "not-an-email"일 수도 있음
  age: number;     // "abc" 또는 -999일 수도 있음
}
```

**올바른 예 (검증용 DTO와 전역 ValidationPipe를 사용함):**

```typescript
// main.ts에서 ValidationPipe 전역 활성화
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,              // 알 수 없는 프로퍼티 제거
      forbidNonWhitelisted: true,   // 제거 대신 알 수 없는 프로퍼티를 거부
      forbidUnknownValues: true,
      validationError: { target: false, value: false },
      transform: true,              // DTO 타입으로 자동 변환
      transformOptions: {
        enableImplicitConversion: false,
      },
    }),
  );

  await app.listen(3000);
}

// 검증을 충분히 적용한 DTO 생성
import {
  IsString,
  IsEmail,
  IsInt,
  Min,
  Max,
  ValidateIf,
  MinLength,
  MaxLength,
  IsNotEmpty,
  IsUUID,
} from 'class-validator';
import { Transform } from 'class-transformer';

export class CreateUserDto {
  @IsString()
  @IsNotEmpty()
  @MinLength(2)
  @MaxLength(100)
  @Transform(({ value }) => typeof value === 'string' ? value.trim() : value)
  name: string;

  @IsEmail()
  @MaxLength(254)
  @Transform(({ value }) => typeof value === 'string' ? value.toLowerCase().trim() : value)
  email: string;

  @IsInt()
  @Min(0)
  @Max(150)
  age: number;

  @IsString()
  @MinLength(12)
  @MaxLength(128)
  password: string;
}

// 빈 문자열·배열·16진수 등을 숫자 0 등으로 조용히 변환하지 않는다.
function toDecimalInteger(value: unknown): unknown {
  return typeof value === 'string' && /^(0|[1-9]\d*)$/.test(value)
    ? Number(value)
    : value;
}

// 기본값과 명시적인 변환을 적용한 쿼리 DTO
export class FindUsersQueryDto {
  @ValidateIf((_, value) => value !== undefined) // null은 허용하지 않는다.
  @IsString()
  @MaxLength(100)
  search?: string;

  @Transform(({ value }) => toDecimalInteger(value))
  @IsInt()
  @Min(1)
  @Max(100)
  limit: number = 20;

  @Transform(({ value }) => toDecimalInteger(value))
  @IsInt()
  @Min(0)
  @Max(10000) // 대량 탐색이 필요하면 커서 기반 페이지네이션 사용
  offset: number = 0;
}

// 경로 매개변수 검증
export class UserIdParamDto {
  @IsUUID('4')
  id: string;
}

@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Post()
  create(@Body() dto: CreateUserDto): Promise<User> {
    // 형태 검증 이후에도 권한·유일성 등 도메인 규칙을 확인한다.
    // 서비스는 password를 해시한 뒤 저장하고 응답 DTO에서 제외해야 한다.
    return this.usersService.create(dto);
  }

  @Get()
  findAll(@Query() query: FindUsersQueryDto): Promise<User[]> {
    // query.limit은 숫자이고 query.search는 타입과 길이를 검증함
    return this.usersService.findAll(query);
  }

  @Get(':id')
  findOne(@Param() params: UserIdParamDto): Promise<User> {
    // params.id는 유효한 UUID
    return this.usersService.findById(params.id);
  }
}
```

DTO는 런타임 클래스여야 하므로 `interface`나 `import type`만으로 전달하지 않는다. 중첩 객체에는 `@ValidateNested()`와 `@Type(() => NestedDto)`, 배열에는 `@IsArray()`·크기 제한·각 원소 검증을 적용한다. `@IsOptional()`은 `undefined`뿐 아니라 `null`도 건너뛰므로 PATCH에서 필드 생략과 명시적인 null을 구분한다.

길이와 비밀번호 정책은 제품 요구와 해시 알고리즘에 맞게 조정한다. 비밀번호를 trim하거나 조용히 잘라서는 안 된다. DTO 검증은 SQL 매개변수 바인딩, 출력 문맥에 맞는 XSS 방어, 객체별 접근 권한 검사를 대체하지 않는다. 요청 파싱 전에 적용되는 본문·파일 크기 제한도 별도로 설정한다.

참고: [NestJS Validation](https://docs.nestjs.com/techniques/validation)
