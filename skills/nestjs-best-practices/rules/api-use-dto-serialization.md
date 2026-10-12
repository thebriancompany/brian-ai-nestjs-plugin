---
title: API 응답에 DTO와 직렬화를 사용한다
impact: MEDIUM
impactDescription: "응답 DTO가 의도하지 않은 데이터 노출을 방지하고 일관성을 보장함"
tags: api, dto, serialization, class-transformer
---

## API 응답에 DTO와 직렬화를 사용한다

민감한 필드를 가진 엔티티를 검증 없이 응답으로 반환하지 않는다. 명시적인 필드 매핑도 유효하며, DTO를 사용한다면 허용 목록 방식으로 직렬화한다. 응답 DTO와 class-transformer의 `@Exclude()`, `@Expose()` 데코레이터로 클라이언트에 보낼 데이터를 정확하게 제어한다. 이렇게 하면 민감한 필드의 우발적인 노출을 방지하고 안정적인 API 계약을 제공할 수 있다.

**잘못된 예 (민감한 필드를 포함한 엔티티를 그대로 반환함):**

```typescript
// 엔티티를 직접 반환함
@Controller('users')
export class UsersController {
  @Get(':id')
  async findOne(@Param('id') id: string): Promise<User> {
    return this.usersService.findById(id);
    // 반환값: { id, email, passwordHash, ssn, internalNotes, ... }
    // 민감한 데이터가 노출됨!
  }
}

// 대안: 명시적인 허용 목록 매핑도 안전한 응답 구성 방식이다.
@Get(':id')
async findOne(@Param('id') id: string) {
  const user = await this.usersService.findById(id);
  return {
    id: user.id,
    email: user.email,
    name: user.name,
    // 필드를 추가할 때 API 계약을 검토하고 응답 테스트를 유지한다.
  };
}
```

**올바른 예 (class-transformer의 @Exclude와 응답 DTO를 사용함):**

```typescript
// class-transformer를 전역으로 활성화
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalInterceptors(new ClassSerializerInterceptor(app.get(Reflector)));
  await app.listen(3000);
}

// 직렬화를 제어하는 엔티티
@Entity()
export class User {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column()
  email: string;

  @Column()
  name: string;

  @Column()
  @Exclude() // 응답에 절대 포함하지 않음
  passwordHash: string;

  @Column({ nullable: true })
  @Exclude()
  ssn: string;

  @Column({ default: false })
  @Exclude({ toPlainOnly: true }) // 응답에서 제외. 쓰기 권한은 별도의 입력 DTO와 인가로 제한한다.
  isAdmin: boolean;

  @CreateDateColumn()
  createdAt: Date;

  @Column()
  @Exclude()
  internalNotes: string;
}

// 기존 제외 목록 방식: 새 민감 필드에 @Exclude를 빠뜨리면 노출된다.
// 새 API에는 아래의 허용 목록 응답 DTO를 우선한다.
@Controller('users')
export class UsersController {
  @Get(':id')
  async findOne(@Param('id') id: string): Promise<User> {
    return this.usersService.findById(id);
    // 반환값: { id, email, name, createdAt }
    // 민감한 필드를 자동으로 제외함
  }
}

// 응답 형태가 다르면 명시적인 DTO를 사용
@Exclude()
export class UserResponseDto {
  @Expose()
  id: string;

  @Expose()
  email: string;

  @Expose()
  name: string;

  @Expose()
  @Transform(({ obj }) => obj.posts?.length ?? 0, { toClassOnly: true })
  postCount: number;

  constructor(partial: Partial<User>) {
    Object.assign(this, partial);
  }
}

@Exclude()
export class PostResponseDto {
  @Expose() id: string;
  @Expose() title: string;
}

@Exclude()
export class UserDetailResponseDto extends UserResponseDto {
  @Expose()
  createdAt: Date;

  @Expose()
  @Type(() => PostResponseDto)
  posts: PostResponseDto[];
}

// 명시적인 DTO를 사용하는 컨트롤러
@Controller('users')
export class UsersController {
  @Get()
  @SerializeOptions({ type: UserResponseDto })
  async findAll(): Promise<UserResponseDto[]> {
    const users = await this.usersService.findAll();
    return users.map(u => plainToInstance(UserResponseDto, u, {
      excludeExtraneousValues: true,
    }));
  }

  @Get(':id')
  async findOne(@Param('id') id: string): Promise<UserDetailResponseDto> {
    const user = await this.usersService.findByIdWithPosts(id);
    return plainToInstance(UserDetailResponseDto, user, {
      excludeExtraneousValues: true,
    });
  }
}

// 중첩 객체도 허용 목록으로 제한한다.
@Exclude()
export class UserSettingsDto {
  @Expose() theme: string;
}

// 조건부 직렬화를 위한 그룹
@Exclude()
export class UserDto {
  @Expose()
  id: string;

  @Expose()
  name: string;

  @Expose({ groups: ['admin'] })
  email: string;

  @Expose({ groups: ['admin'] })
  createdAt: Date;

  @Expose({ groups: ['admin', 'owner'] })
  @Type(() => UserSettingsDto)
  settings: UserSettingsDto;
}

@Controller('users')
export class UsersController {
  @Get()
  @SerializeOptions({ groups: ['public'] })
  async findAllPublic(): Promise<UserDto[]> {
    // 반환값: { id, name }
  }

  @Get('admin')
  @UseGuards(AdminGuard)
  @SerializeOptions({ groups: ['admin'] })
  async findAllAdmin(): Promise<UserDto[]> {
    // 반환값: { id, name, email, createdAt }
  }

  @Get('me')
  @SerializeOptions({ groups: ['owner'] })
  async getProfile(@CurrentUser() user: User): Promise<UserDto> {
    // 반환값: { id, name, settings }
  }
}
```

참고: [NestJS Serialization](https://docs.nestjs.com/techniques/serialization)
