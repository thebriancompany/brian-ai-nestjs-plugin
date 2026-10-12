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

**올바른 예 (하나의 책임에 집중하는 서비스):**

```typescript
// 단일 책임에 집중하는 서비스
@Injectable()
export class UsersService {
  constructor(private userRepo: UserRepository) {}

  async create(dto: CreateUserDto): Promise<User> {
    return this.userRepo.save(dto);
  }

  async findById(id: string): Promise<User> {
    return this.userRepo.findOneOrFail({ where: { id } });
  }
}

@Injectable()
export class OrdersService {
  constructor(private orderRepo: OrderRepository) {}

  async create(userId: string, dto: CreateOrderDto): Promise<Order> {
    return this.orderRepo.save({ userId, ...dto });
  }

  async findByUser(userId: string): Promise<Order[]> {
    return this.orderRepo.find({ where: { userId } });
  }
}

@Injectable()
export class OrderStatsService {
  constructor(private orderRepo: OrderRepository) {}

  async calculateForUser(userId: string): Promise<OrderStats> {
    // 통계 계산에 집중
  }
}

// 컨트롤러나 전용 조정 서비스에서 작업 흐름을 조율
@Controller('orders')
export class OrdersController {
  constructor(
    private orders: OrdersService,
    private payment: PaymentService,
    private notifications: NotificationService,
  ) {}

  @Post()
  async create(@CurrentUser() user: User, @Body() dto: CreateOrderDto) {
    const order = await this.orders.create(user.id, dto);
    await this.payment.charge(order);
    await this.notifications.sendOrderConfirmation(order);
    return order;
  }
}
```

참고: [NestJS Providers](https://docs.nestjs.com/providers)
