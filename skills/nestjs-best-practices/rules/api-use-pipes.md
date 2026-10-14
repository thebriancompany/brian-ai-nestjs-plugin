---
title: 입력 변환에 파이프를 사용한다
impact: MEDIUM
impactDescription: "검증한 입력을 예측 가능한 타입으로 핸들러에 전달함"
tags: api, pipes, validation, transformation
---

## 입력 변환에 파이프를 사용한다

단일 값은 `ParseIntPipe`, `ParseUUIDPipe` 같은 내장 파이프로 변환하고, 여러 필드의 범위·필수 조건은 DTO와 `ValidationPipe`로 검증한다. 숫자로 바뀌었다고 유효한 페이지 크기인 것은 아니다. 정규화와 검증도 별개의 단계다.

**잘못된 예 (파싱 성공을 유효성 검증으로 간주함):**

```typescript
@Get()
findAll(@Query('page') page: string, @Query('limit') limit: string) {
  return this.usersService.findAll(parseInt(page) || 1, parseInt(limit) || 10);
  // '12abc'를 12로 허용하고 음수·과도한 limit도 통과한다.
}
```

**올바른 예 (변환 후 범위와 형태를 검증함):**

```typescript
app.useGlobalPipes(new ValidationPipe({
  whitelist: true,
  forbidNonWhitelisted: true,
  transform: true,
  transformOptions: { enableImplicitConversion: false },
}));

export class FindProductsDto {
  @Type(() => Number)
  @IsInt()
  @Min(1)
  @Max(10_000)
  page: number = 1;

  @Type(() => Number)
  @IsInt()
  @Min(1)
  @Max(100)
  limit: number = 10;

  @IsOptional()
  @Transform(({ value }) => typeof value === 'string' ? value.trim() : value)
  @IsString()
  @MaxLength(100)
  search?: string;

  @IsOptional()
  @Transform(({ value }) => typeof value === 'string'
    ? value.split(',').map(item => item.trim())
    : value)
  @IsArray()
  @ArrayMaxSize(20)
  @ArrayUnique()
  @IsUUID('4', { each: true })
  categoryIds?: string[];
}

@Controller('products')
export class ProductsController {
  constructor(private readonly productsService: ProductsService) {}

  @Get()
  findAll(@Query() query: FindProductsDto): Promise<ProductResponseDto[]> {
    return this.productsService.findAll(query);
  }

  @Get(':id')
  findOne(@Param('id', new ParseUUIDPipe({ version: '4' })) id: string) {
    return this.productsService.findOne(id);
  }
}
```

DTO는 `interface`나 `import type`이 아닌 런타임 클래스여야 한다. 누락 시 기본값을 쓰는 필드에 `@IsOptional()`을 붙이면 `null`까지 통과시킬 수 있으므로 필드 계약에 맞춰 선택한다. 서비스도 정렬 순서를 고정하고 검증한 `limit`을 실제 조회에 적용한다. 큰 목록에는 커서 페이지네이션을 고려한다.

날짜처럼 앱 계약이 정해진 값은 전용 파이프로 처리한다. 아래 계약은 `YYYY-MM-DD` 달력 날짜를 UTC 자정으로 변환한다. 지역 시간의 일자 조회라면 서비스에서 해당 시간대의 시작·끝을 계산한다.

```typescript
@Injectable()
export class ParseCalendarDatePipe implements PipeTransform<unknown, Date> {
  transform(value: unknown): Date {
    if (typeof value !== 'string' || !/^\d{4}-\d{2}-\d{2}$/.test(value)) {
      throw new BadRequestException('Expected YYYY-MM-DD');
    }
    const date = new Date(`${value}T00:00:00.000Z`);
    if (!Number.isFinite(date.getTime()) || date.toISOString().slice(0, 10) !== value) {
      throw new BadRequestException('Invalid calendar date');
    }
    return date;
  }
}

@Get('reports')
getReports(
  @Query('from', ParseCalendarDatePipe) from: Date,
  @Query('to', ParseCalendarDatePipe) to: Date,
) {
  if (from >= to) throw new BadRequestException('from must be before to');
  // [from, to) 범위. 최대 조회 기간도 업무 계약에 맞게 제한한다.
  return this.reportsService.findBetween(from, to);
}
```

형식 오류에는 기본 400 응답을 유지한다. 파이프에서 임의 HTML 제거·이메일 소문자화 등을 모든 입력에 적용하지 않는다. 저장할 값과 렌더링할 문맥에 따라 [입력 검증](security-validate-all-input.md), [출력 처리](security-sanitize-output.md) 규칙을 적용한다.

참고: [NestJS 11 Pipes](https://docs.nestjs.com/v11/pipes), [class-validator](https://github.com/typestack/class-validator)
