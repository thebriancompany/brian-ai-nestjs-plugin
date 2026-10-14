---
title: 서비스에 단일 책임을 부여한다
impact: CRITICAL
impactDescription: "독립적인 책임을 분리해 테스트 범위를 명확히 함"
tags: architecture, services, single-responsibility
---

## 서비스에 단일 책임을 부여한다

각 서비스는 명확하게 정의된 하나의 책임을 가져야 한다. 관련 없는 여러 관심사를 처리하는 거대 서비스인 "god service"를 피한다. 이름이나 메서드 개수만으로 위반을 판단하지 않는다. 서로 다른 이유로 변경되는 책임을 분리하고, 하나의 업무 흐름을 조율하는 서비스는 필요한 협력을 유지한다.

**잘못된 예 (거대 서비스 안티패턴):**

```typescript
// 거대 서비스 안티패턴
@Injectable()
export class UserAndOrderService {
  constructor(
    private userRepo: UserRepository,
    private orderRepo: OrderRepository,
    private mailer: MailService,
    private payment: PaymentService,
  ) {}

  async createUser(dto: CreateUserDto) {
    const user = await this.userRepo.save(dto);
    await this.mailer.sendWelcome(user);
    return user;
  }

  async createOrder(userId: string, dto: CreateOrderDto) {
    const order = await this.orderRepo.save({ userId, ...dto });
    await this.payment.charge(order);
    await this.mailer.sendOrderConfirmation(order);
    return order;
  }

  async calculateOrderStats(userId: string) {
    // 통계 로직이 섞여 있음
  }

  async validatePayment(orderId: string) {
    // 결제 로직이 섞여 있음
  }
}
```

**올바른 예 (기능의 책임을 분리하고 조율은 별도 서비스에 둠):**

```typescript
// 단일 책임에 집중하는 서비스
@Injectable()
export class UsersService {
  constructor(private readonly userRepo: UserRepository) {}

  async findById(id: string): Promise<User> {
    const user = await this.userRepo.findOne({ where: { id } });
    if (!user) throw new NotFoundException('User not found');
    return user;
  }
}

@Injectable()
export class OrdersService {
  constructor(private readonly orderRepo: OrderRepository) {}

  async findRecentByUser(userId: string): Promise<Order[]> {
    return this.orderRepo.find({
      where: { userId },
      order: { createdAt: 'DESC', id: 'DESC' },
      take: 20,
    });
  }
}

@Injectable()
export class UserOverviewService {
  constructor(
    private readonly users: UsersService,
    private readonly orders: OrdersService,
  ) {}

  async getOverview(userId: string) {
    const user = await this.users.findById(userId);
    const recentOrders = await this.orders.findRecentByUser(user.id);
    // 실제 응답은 출력 DTO로 필요한 필드만 직렬화한다.
    return { userId: user.id, recentOrders };
  }
}

// HTTP 계층은 인증된 사용자와 입력을 전달하고 결과를 반환
// CurrentUser는 인증 가드가 설정한 사용자를 읽는 프로젝트 데코레이터다.
@Controller('me')
export class MyOverviewController {
  constructor(private readonly overview: UserOverviewService) {}

  @Get('overview')
  getOverview(@CurrentUser() user: User) {
    return this.overview.getOverview(user.id);
  }
}
```

각 기능 모듈은 필요한 서비스를 내보내고, 조정 서비스의 모듈은 해당 기능 모듈들을 가져온다. 같은 업무 흐름에 속하는 메서드를 개별 클래스로 무조건 나누지는 않는다. 주문·결제처럼 여러 쓰기를 수행하는 흐름은 조정 서비스가 트랜잭션, 재시도, 멱등성, 실패 후 상태를 책임져야 한다. 메서드를 여러 서비스로 옮기기만 해서는 원자성이 생기지 않는다.

참고: [NestJS Providers](https://docs.nestjs.com/providers)
