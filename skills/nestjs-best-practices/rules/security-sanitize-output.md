---
title: 출력 문맥에 맞게 XSS를 방지한다
impact: HIGH
impactDescription: "HTML 실행 지점에서 사용자 콘텐츠를 안전하게 처리함"
tags: security, xss, sanitization, html
---

## 출력 문맥에 맞게 XSS를 방지한다

일반 텍스트는 그대로 저장하고 HTML에 표시할 때 템플릿의 자동 이스케이프나 DOM의 `textContent`를 사용한다. 사용자에게 HTML 작성을 허용하는 필드만 허용 목록 기반 정제기를 적용한다. JSON 응답에 문자열이 들어 있다는 사실 자체가 XSS는 아니지만, 브라우저에서 그 값을 `innerHTML` 등으로 실행하면 문제가 된다. HTML 본문용 정제 결과도 JavaScript·CSS·URL·HTML 속성 문맥에 그대로 넣어서는 안 된다.

**잘못된 예 (사용자 HTML을 실행 가능한 문맥에 그대로 삽입함):**

```typescript
@Controller('pages')
export class PagesController {
  @Get(':slug')
  @Header('Content-Type', 'text/html; charset=utf-8')
  async getPage(@Param('slug') slug: string): Promise<string> {
    const page = await this.pagesService.findBySlug(slug);
    // <script>, 이벤트 속성, javascript: 링크 등이 브라우저에서 실행될 수 있다.
    return `<html><body>${page.content}</body></html>`;
  }
}
```

**올바른 예 (허용된 HTML만 본문에 렌더링함):**

```typescript
// esModuleInterop을 사용하는 TypeScript 설정 기준
import sanitizeHtml from 'sanitize-html';
import { IsString, MaxLength } from 'class-validator';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';

const richTextPolicy: sanitizeHtml.IOptions = {
  allowedTags: ['b', 'i', 'em', 'strong', 'a', 'p', 'br'],
  allowedAttributes: { a: ['href', 'title'] },
  allowedSchemes: ['http', 'https', 'mailto'],
  allowProtocolRelative: false,
};

export class CreateCommentDto {
  @IsString()
  @MaxLength(10000)
  content: string;
}

@Injectable()
export class CommentsService {
  constructor(@InjectRepository(Comment) private readonly repo: Repository<Comment>) {}

  // authorId는 인증 가드의 현재 사용자에서 전달한다. 요청 본문에서 받지 않는다.
  async create(dto: CreateCommentDto, authorId: string): Promise<Comment> {
    return this.repo.save({
      content: sanitizeHtml(dto.content, richTextPolicy),
      authorId,
    });
  }
}

@Controller('pages')
export class PagesController {
  constructor(private readonly pagesService: PagesService) {}

  @Get(':slug')
  @Header('Content-Type', 'text/html; charset=utf-8')
  async getPage(@Param('slug') slug: string): Promise<string> {
    const page = await this.pagesService.findBySlug(slug);
    // 기존 데이터나 다른 저장 경로도 현재 정책을 거치도록 출력 시 적용한다.
    const content = sanitizeHtml(page.content, richTextPolicy);
    return `<!doctype html><html><body>${content}</body></html>`;
  }
}

// Express 어댑터: 보안 헤더는 HTML 정제와 함께 사용하는 방어 수단이다.
import helmet from 'helmet';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.use(helmet()); // 기본 CSP를 시작점으로 실제 렌더링 정책에 맞게 조정한다.
  await app.listen(3000);
}
```

정제 후 문자열에 검증되지 않은 HTML을 덧붙이면 보호가 깨진다. 클라이언트도 일반 텍스트에는 자동 이스케이프를 유지하고, HTML 렌더링을 허용하는 위치를 한정한다. API의 CSP 헤더는 별도 프런트엔드 문서에 적용되지 않는다. Fastify에서는 Express용 `helmet()` 미들웨어 대신 호환되는 `@fastify/helmet` 플러그인을 등록한다.

DTO의 길이 제한과 별도로 HTTP 파서의 본문 크기 제한도 설정한다. HTML을 제거하는 것으로 SQL 인젝션이나 소유자 위조를 방지할 수는 없으며, DB 매개변수 바인딩과 인증·인가를 별도로 적용한다. JSON 응답은 Nest의 기본 직렬화를 사용하고 비밀번호 해시 등 민감한 필드는 [응답 DTO 규칙](api-use-dto-serialization.md)에 따라 제외한다.

참고: [OWASP XSS Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html), [sanitize-html](https://github.com/apostrophecms/sanitize-html), [NestJS 보안 헤더](https://docs.nestjs.com/security/helmet)
