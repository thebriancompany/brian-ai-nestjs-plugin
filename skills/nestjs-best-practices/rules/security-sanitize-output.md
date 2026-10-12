---
title: 출력을 정제해 XSS를 방지한다
impact: HIGH
impactDescription: "XSS 취약점은 사용자 세션과 데이터를 침해할 수 있음"
tags: security, xss, sanitization, html
---

## 출력을 정제해 XSS를 방지한다

NestJS API는 대체로 브라우저가 실행하지 않는 JSON을 반환하지만, HTML을 렌더링하거나 사용자 콘텐츠를 저장하거나 프런트엔드 프레임워크가 API 응답을 잘못 처리하면 XSS 위험이 발생한다. 사용자가 만든 콘텐츠는 저장 전에 정제하고 적절한 Content-Type 헤더를 사용한다.

**잘못된 예 (정제하지 않은 HTML을 그대로 저장함):**

```typescript
// 사용자가 보낸 HTML을 그대로 저장
@Injectable()
export class CommentsService {
  async create(dto: CreateCommentDto): Promise<Comment> {
    // 사용자가 다음 코드를 주입할 수 있음: <script>steal(document.cookie)</script>
    return this.repo.save({
      content: dto.content, // 정제하지 않은 원본
      authorId: dto.authorId,
    });
  }
}

// 정제하지 않고 HTML 반환
@Controller('pages')
export class PagesController {
  @Get(':slug')
  @Header('Content-Type', 'text/html')
  async getPage(@Param('slug') slug: string): Promise<string> {
    const page = await this.pagesService.findBySlug(slug);
    // page.content에 사용자 입력이 있으면 XSS 가능
    return `<html><body>${page.content}</body></html>`;
  }
}

// 오류에 사용자 입력 반영
@Get(':id')
async findOne(@Param('id') id: string): Promise<User> {
  const user = await this.repo.findOne({ where: { id } });
  if (!user) {
    // id에 악성 콘텐츠가 있고 오류를 렌더링하면 XSS 발생
    throw new NotFoundException(`User ${id} not found`);
  }
  return user;
}
```

**올바른 예 (콘텐츠를 정제하고 적절한 헤더를 사용함):**

```typescript
// 저장 전에 HTML 콘텐츠 정제
import * as sanitizeHtml from 'sanitize-html';

@Injectable()
export class CommentsService {
  private readonly sanitizeOptions: sanitizeHtml.IOptions = {
    allowedTags: ['b', 'i', 'em', 'strong', 'a', 'p', 'br'],
    allowedAttributes: {
      a: ['href', 'title'],
    },
    allowedSchemes: ['http', 'https', 'mailto'],
  };

  async create(dto: CreateCommentDto): Promise<Comment> {
    return this.repo.save({
      content: sanitizeHtml(dto.content, this.sanitizeOptions),
      authorId: dto.authorId,
    });
  }
}

// 검증 파이프로 HTML 제거
import { Transform } from 'class-transformer';

export class CreatePostDto {
  @IsString()
  @MaxLength(1000)
  @Transform(({ value }) => typeof value === 'string' ? sanitizeHtml(value, { allowedTags: [] }) : value)
  title: string;

  @IsString()
  @Transform(({ value }) =>
    typeof value === 'string' ? sanitizeHtml(value, {
      allowedTags: ['p', 'br', 'b', 'i', 'a'],
      allowedAttributes: { a: ['href'] },
    }) : value,
  )
  content: string;
}

// 적절한 Content-Type 헤더 설정
@Controller('api')
export class ApiController {
  @Get('data')
  @Header('Content-Type', 'application/json')
  async getData(): Promise<DataResponse> {
    // JSON 응답이므로 브라우저가 스크립트를 실행하지 않음
    return this.service.getData();
  }
}

// 오류 메시지 정제
@Get(':id')
async findOne(@Param('id', ParseUUIDPipe) id: string): Promise<User> {
  const user = await this.repo.findOne({ where: { id } });
  if (!user) {
    // UUID 검증으로 안전한 형식 보장
    throw new NotFoundException('User not found');
  }
  return user;
}

// Helmet으로 CSP 헤더 설정
import helmet from 'helmet';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.use(
    helmet({
      contentSecurityPolicy: {
        directives: {
          defaultSrc: ["'self'"],
          scriptSrc: ["'self'"],
          styleSrc: ["'self'", "'unsafe-inline'"],
          imgSrc: ["'self'", 'data:', 'https:'],
        },
      },
    }),
  );

  await app.listen(3000);
}
```

참고: [OWASP XSS Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
