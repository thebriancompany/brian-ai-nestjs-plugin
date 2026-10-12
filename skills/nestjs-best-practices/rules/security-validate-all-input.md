---
title: DTO와 파이프로 모든 입력을 검증한다
impact: HIGH
impactDescription: "공격에 대한 첫 번째 방어선임"
tags: security, validation, dto, pipes
---

## DTO와 파이프로 모든 입력을 검증한다

항상 DTO의 class-validator 데코레이터와 전역 ValidationPipe로 들어오는 데이터를 검증한다. 사용자 입력을 신뢰하지 않는다. 처리 전에 모든 요청 본문, 쿼리 매개변수, 경로 매개변수를 검증한다.

**잘못된 예 (검증 없이 원시 입력을 신뢰함):**

```typescript
// 검증 없이 원시 입력 신뢰
@Controller('users')
export class UsersController {
  @Post()
  create(@Body() body: any) {
    // body에는 SQL 인젝션, XSS 등 무엇이든 들어올 수 있음
    return this.usersService.create(body);
  }

  @Get()
  findAll(@Query() query: any) {
    // query.limit은 "'; DROP TABLE users; --"일 수도 있음
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
      forbidNonWhitelisted: true,   // 알 수 없는 프로퍼티에 예외 발생
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
  IsOptional,
  MinLength,
  MaxLength,
  Matches,
  IsNotEmpty,
  IsUUID,
} from 'class-validator';
import { Transform, Type } from 'class-transformer';

export class CreateUserDto {
  @IsString()
  @IsNotEmpty()
  @MinLength(2)
  @MaxLength(100)
  @Transform(({ value }) => typeof value === 'string' ? value.trim() : value)
  name: string;

  @IsEmail()
  @Transform(({ value }) => typeof value === 'string' ? value.toLowerCase().trim() : value)
  email: string;

  @IsInt()
  @Min(0)
  @Max(150)
  age: number;

  @IsString()
  @MinLength(8)
  @MaxLength(100)
  @Matches(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/, {
    message: 'Password must contain uppercase, lowercase, and number',
  })
  password: string;
}

// 기본값과 변환을 적용한 쿼리 DTO
export class FindUsersQueryDto {
  @IsOptional()
  @IsString()
  @MaxLength(100)
  search?: string;

  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  @Max(100)
  limit: number = 20;

  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(0)
  offset: number = 0;
}

// 경로 매개변수 검증
export class UserIdParamDto {
  @IsUUID('4')
  id: string;
}

@Controller('users')
export class UsersController {
  @Post()
  create(@Body() dto: CreateUserDto): Promise<User> {
    // dto가 유효함을 보장함
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

참고: [NestJS Validation](https://docs.nestjs.com/techniques/validation)
