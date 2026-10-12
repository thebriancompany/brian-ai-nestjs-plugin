---
title: 입력 변환에 파이프를 사용한다
impact: MEDIUM
impactDescription: "정제하고 검증한 데이터를 핸들러에 전달함"
tags: api, pipes, validation, transformation
---

## 입력 변환에 파이프를 사용한다

일반적인 변환에는 `ParseIntPipe`, `ParseUUIDPipe`, `DefaultValuePipe` 같은 내장 파이프를 사용한다. 비즈니스에 특화된 변환은 사용자 정의 파이프로 구현한다. 파이프는 검증과 변환 로직을 컨트롤러에서 분리한다.

**잘못된 예 (핸들러에서 수동으로 타입을 파싱함):**

```typescript
// 핸들러에서 수동으로 타입 파싱
@Controller('users')
export class UsersController {
  @Get(':id')
  async findOne(@Param('id') id: string): Promise<User> {
    // 모든 핸들러에서 수동 검증
    const uuid = id.trim();
    if (!isUUID(uuid)) {
      throw new BadRequestException('Invalid UUID');
    }
    return this.usersService.findOne(uuid);
  }

  @Get()
  async findAll(
    @Query('page') page: string,
    @Query('limit') limit: string,
  ): Promise<User[]> {
    // 수동 파싱과 기본값 처리
    const pageNum = parseInt(page) || 1;
    const limitNum = parseInt(limit) || 10;
    return this.usersService.findAll(pageNum, limitNum);
  }
}

// 검증 없는 타입 변환
@Get()
async search(@Query('price') price: string): Promise<Product[]> {
  const priceNum = +price; // 유효하지 않으면 NaN이며 오류는 발생하지 않음
  return this.productsService.findByPrice(priceNum);
}
```

**올바른 예 (내장 파이프와 사용자 정의 파이프를 사용함):**

```typescript
// 일반적인 변환에는 내장 파이프 사용
@Controller('users')
export class UsersController {
  @Get(':id')
  async findOne(@Param('id', ParseUUIDPipe) id: string): Promise<User> {
    // id가 유효한 UUID임을 보장함
    return this.usersService.findOne(id);
  }

  @Get()
  async findAll(
    @Query('page', new DefaultValuePipe(1), ParseIntPipe) page: number,
    @Query('limit', new DefaultValuePipe(10), ParseIntPipe) limit: number,
  ): Promise<User[]> {
    // 자동 기본값 적용과 타입 변환
    return this.usersService.findAll(page, limit);
  }

  @Get('by-status/:status')
  async findByStatus(
    @Param('status', new ParseEnumPipe(UserStatus)) status: UserStatus,
  ): Promise<User[]> {
    return this.usersService.findByStatus(status);
  }
}

// 비즈니스 로직을 위한 사용자 정의 파이프
@Injectable()
export class ParseDatePipe implements PipeTransform<string, Date> {
  transform(value: string): Date {
    const date = new Date(value);
    if (isNaN(date.getTime())) {
      throw new BadRequestException('Invalid date format');
    }
    return date;
  }
}

@Get('reports')
async getReports(
  @Query('from', ParseDatePipe) from: Date,
  @Query('to', ParseDatePipe) to: Date,
): Promise<Report[]> {
  return this.reportsService.findBetween(from, to);
}

// 사용자 정의 변환 파이프
@Injectable()
export class NormalizeEmailPipe implements PipeTransform<string, string> {
  transform(value: string): string {
    if (!value) return value;
    return value.trim().toLowerCase();
  }
}

// 쉼표로 구분된 값 파싱
@Injectable()
export class ParseArrayPipe implements PipeTransform<string, string[]> {
  transform(value: string): string[] {
    if (!value) return [];
    return value.split(',').map((v) => v.trim()).filter(Boolean);
  }
}

@Get('products')
async findProducts(
  @Query('ids', ParseArrayPipe) ids: string[],
  @Query('email', NormalizeEmailPipe) email: string,
): Promise<Product[]> {
  // ids는 이미 배열이고 email은 정규화됨
  return this.productsService.findByIds(ids);
}

// HTML 입력 정제
@Injectable()
export class SanitizeHtmlPipe implements PipeTransform<string, string> {
  transform(value: string): string {
    if (typeof value !== 'string') throw new BadRequestException('Expected string');
    return sanitizeHtml(value, { allowedTags: [] });
  }
}

// 변환 기능을 갖춘 전역 검증 파이프
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true, // DTO에 없는 프로퍼티 제거
    transform: true, // DTO 타입으로 자동 변환
    transformOptions: {
      enableImplicitConversion: false, // 숫자 변환은 DTO의 @Type(() => Number)로 명시
    },
    forbidNonWhitelisted: true, // 추가 프로퍼티가 있으면 예외 발생
  }),
);

// 변환 데코레이터를 적용한 DTO
export class FindProductsDto {
  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  page?: number = 1;

  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  @Max(100)
  limit?: number = 10;

  @IsOptional()
  @Transform(({ value }) => typeof value === 'string' ? value.toLowerCase() : value)
  @IsString()
  search?: string;

  @IsOptional()
  @Transform(({ value }) => {
    if (typeof value === 'string') return value.split(',').map(v => v.trim());
    return value; // 배열은 그대로 검증하고, 객체·숫자 등은 @IsArray로 거부한다.
  })
  @IsArray()
  @IsString({ each: true })
  categories?: string[];
}

@Get()
async findAll(@Query() dto: FindProductsDto): Promise<Product[]> {
  // dto는 이미 변환하고 검증한 상태
  return this.productsService.findAll(dto);
}

// 파이프 오류 사용자 정의
@Injectable()
export class CustomParseIntPipe extends ParseIntPipe {
  constructor() {
    super({
      exceptionFactory: (error) =>
        new BadRequestException(`${error} must be a valid integer`),
    });
  }
}

// 또는 내장 파이프 옵션 사용
@Get(':id')
async findOne(
  @Param(
    'id',
    new ParseIntPipe({
      errorHttpStatusCode: HttpStatus.NOT_ACCEPTABLE,
      exceptionFactory: () => new NotAcceptableException('ID must be numeric'),
    }),
  )
  id: number,
): Promise<Item> {
  return this.itemsService.findOne(id);
}
```

참고: [NestJS Pipes](https://docs.nestjs.com/pipes)
