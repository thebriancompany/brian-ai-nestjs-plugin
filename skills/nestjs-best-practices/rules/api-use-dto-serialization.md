---
title: API 응답에 DTO와 직렬화를 사용한다
impact: HIGH
impactDescription: "허용 목록 응답으로 민감한 필드의 우발적 노출을 방지함"
tags: api, dto, serialization, class-transformer
---

## API 응답에 DTO와 직렬화를 사용한다

반환 타입 선언만으로 응답 필드가 제거되지는 않는다. 엔티티 전체를 반환하지 말고 명시적으로 공개 필드를 매핑하거나 허용 목록 DTO로 변환한다. `@Exclude()`를 민감한 필드에만 붙이는 방식은 새 필드가 추가될 때 누락되기 쉽다.

**잘못된 예 (반환 타입만 바꾸면 필드가 제거된다고 가정함):**

```typescript
@Get(':id')
async findOne(@Param('id') id: string): Promise<UserResponseDto> {
  return this.usersService.findById(id);
  // 실제 객체의 passwordHash, internalNotes 등도 응답에 남는다.
}
```

**올바른 예 (명시적인 허용 목록으로 응답을 구성함):**

작은 응답에는 다음 매핑만으로 충분하다. 조회 대상에 대한 접근 권한은 서비스나 가드에서 별도로 검사한다.

```typescript
@Get(':id')
async findOne(@Param('id', ParseUUIDPipe) id: string) {
  const user = await this.usersService.findById(id);
  if (!user) throw new NotFoundException('User not found');
  return { id: user.id, name: user.name };
}
```

공유 DTO와 중첩 응답이 필요하면 `class-transformer`를 사용한다. 아래는 별도의 대안이며 앞의 매핑과 함께 적용할 필요는 없다.

```typescript
import { Exclude, Expose, Type, plainToInstance } from 'class-transformer';

@Exclude()
export class PostResponseDto {
  @Expose() id!: string;
  @Expose() title!: string;
}

@Exclude()
export class UserResponseDto {
  @Expose() id!: string;
  @Expose() name!: string;
}

@Exclude()
export class UserDetailResponseDto extends UserResponseDto {
  @Expose() createdAt!: Date;
  @Expose() postCount!: number;
  @Expose()
  @Type(() => PostResponseDto)
  posts!: PostResponseDto[];
}

@Controller('users')
@UseInterceptors(ClassSerializerInterceptor)
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Get()
  async findAll(): Promise<UserResponseDto[]> {
    // 서비스는 검증한 페이지 크기로 조회 범위를 제한한다.
    const users = await this.usersService.findFirstPage();
    return plainToInstance(UserResponseDto, users, { excludeExtraneousValues: true });
  }

  @Get(':id')
  async findOne(@Param('id', ParseUUIDPipe) id: string): Promise<UserDetailResponseDto> {
    // 이 예제의 posts는 의도적으로 로드한 공개 게시물이다.
    const user = await this.usersService.findByIdWithPublicPosts(id);
    if (!user) throw new NotFoundException('User not found');
    return plainToInstance(UserDetailResponseDto, {
      id: user.id,
      name: user.name,
      createdAt: user.createdAt,
      posts: user.posts,
      postCount: user.posts.length,
    }, { excludeExtraneousValues: true });
  }
}
```

중첩 객체에도 DTO와 `@Type`을 적용한다. `postCount`가 전체 개수라면 페이지에 포함된 배열 길이 대신 별도 집계 결과를 전달한다. `new Dto()` 없이 원시 객체를 반환하거나 `{ data: entity }`로 감싼 경우에는 DTO의 직렬화 규칙이 자동으로 적용된다고 가정하지 않는다. 페이지 응답에도 중첩 DTO를 명시한다.

직렬화 그룹은 출력 필드 선택 기능이며 인가를 대신하지 않는다. 관리자용 그룹은 인가를 통과한 서버 코드에서만 선택한다. 테스트에는 새 민감 필드와 중첩 민감 필드가 응답에서 빠지는지도 포함한다. 입력 DTO는 별도로 정의해 수정 가능한 필드를 제한한다.

참고: [NestJS Serialization](https://docs.nestjs.com/techniques/serialization), [class-transformer](https://github.com/typestack/class-transformer)
