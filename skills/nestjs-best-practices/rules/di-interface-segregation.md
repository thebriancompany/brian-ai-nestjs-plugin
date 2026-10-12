---
title: 인터페이스 분리 원칙을 적용한다
impact: HIGH
impactDescription: "사용하지 않는 계약에 대한 의존성을 줄임"
tags: dependency-injection, interfaces, solid, isp
---

## 인터페이스 분리 원칙을 적용한다

사용하는 쪽이 필요하지 않은 인터페이스에 의존하도록 강요해서는 안 된다. NestJS에서는 관련 없는 메서드를 묶은 거대한 인터페이스 대신 특정 기능에 집중하는 작은 인터페이스를 유지하라는 뜻이다. 이메일 전송만 필요한 서비스가 SMS, 푸시 알림, 로깅까지 포함한 인터페이스에 의존해서는 안 된다. 큰 인터페이스를 역할별로 분리한다.

**잘못된 예 (거대한 인터페이스가 사용하지 않는 의존성까지 강제함):**

```typescript
// 거대한 인터페이스가 모든 사용처에 모든 기능에 대한 의존을 강제함
interface NotificationService {
  sendEmail(to: string, subject: string, body: string): Promise<void>;
  sendSms(phone: string, message: string): Promise<void>;
  sendPush(userId: string, notification: PushPayload): Promise<void>;
  sendSlack(channel: string, message: string): Promise<void>;
  logNotification(type: string, payload: any): Promise<void>;
  getDeliveryStatus(id: string): Promise<DeliveryStatus>;
  retryFailed(id: string): Promise<void>;
  scheduleNotification(dto: ScheduleDto): Promise<string>;
}

// 이메일만 필요하지만 테스트에서 모든 기능을 모의 구현해야 함
@Injectable()
export class OrdersService {
  constructor(
    private notifications: NotificationService, // 8개 메서드에 의존하지만 1개만 사용
  ) {}

  async confirmOrder(order: Order): Promise<void> {
    await this.notifications.sendEmail(
      order.customer.email,
      'Order Confirmed',
      `Your order ${order.id} has been confirmed.`,
    );
  }
}

// 사용하지 않는 메서드까지 모의 구현해야 하므로 테스트가 어려움
const mockNotificationService = {
  sendEmail: jest.fn(),
  sendSms: jest.fn(),           // 사용하지 않지만 필수
  sendPush: jest.fn(),          // 사용하지 않지만 필수
  sendSlack: jest.fn(),         // 사용하지 않지만 필수
  logNotification: jest.fn(),   // 사용하지 않지만 필수
  getDeliveryStatus: jest.fn(), // 사용하지 않지만 필수
  retryFailed: jest.fn(),       // 사용하지 않지만 필수
  scheduleNotification: jest.fn(), // 사용하지 않지만 필수
};
```

**올바른 예 (기능별로 인터페이스를 분리함):**

```typescript
// 분리한 인터페이스: 각각 하나의 기능에 집중
interface EmailSender {
  sendEmail(to: string, subject: string, body: string): Promise<void>;
}

interface SmsSender {
  sendSms(phone: string, message: string): Promise<void>;
}

interface PushSender {
  sendPush(userId: string, notification: PushPayload): Promise<void>;
}

interface NotificationLogger {
  logNotification(type: string, payload: any): Promise<void>;
}

interface NotificationScheduler {
  scheduleNotification(dto: ScheduleDto): Promise<string>;
}

// 하나의 구현체가 여러 인터페이스를 구현할 수 있음
@Injectable()
export class NotificationService implements EmailSender, SmsSender, PushSender {
  async sendEmail(to: string, subject: string, body: string): Promise<void> {
    // 이메일 구현
  }

  async sendSms(phone: string, message: string): Promise<void> {
    // SMS 구현
  }

  async sendPush(userId: string, notification: PushPayload): Promise<void> {
    // 푸시 구현
  }
}

// 또는 구현체를 분리
@Injectable()
export class SendGridEmailService implements EmailSender {
  async sendEmail(to: string, subject: string, body: string): Promise<void> {
    // SendGrid 전용 구현
  }
}

// 사용하는 쪽은 필요한 기능에만 의존
@Injectable()
export class OrdersService {
  constructor(
    @Inject(EMAIL_SENDER) private emailSender: EmailSender, // 최소 의존성
  ) {}

  async confirmOrder(order: Order): Promise<void> {
    await this.emailSender.sendEmail(
      order.customer.email,
      'Order Confirmed',
      `Your order ${order.id} has been confirmed.`,
    );
  }
}

// 사용하는 기능만 모의 구현하면 되므로 테스트가 간단함
const mockEmailSender: EmailSender = {
  sendEmail: jest.fn(),
};

// 토큰을 사용한 모듈 등록
export const EMAIL_SENDER = Symbol('EMAIL_SENDER');
export const SMS_SENDER = Symbol('SMS_SENDER');

@Module({
  providers: [
    { provide: EMAIL_SENDER, useClass: SendGridEmailService },
    { provide: SMS_SENDER, useClass: TwilioSmsService },
  ],
  exports: [EMAIL_SENDER, SMS_SENDER],
})
export class NotificationModule {}
```

**필요할 때 인터페이스를 조합하기:**

```typescript
// 사용하는 쪽에 실제로 여러 기능이 필요한 경우도 있음
interface EmailAndSmsSender extends EmailSender, SmsSender {}

// 또는 교차 타입 사용
type MultiChannelSender = EmailSender & SmsSender & PushSender;

// 여러 채널이 필요한 서비스
@Injectable()
export class AlertService {
  constructor(
    @Inject(MULTI_CHANNEL_SENDER)
    private sender: EmailSender & SmsSender,
  ) {}

  async sendCriticalAlert(user: User, message: string): Promise<void> {
    await Promise.all([
      this.sender.sendEmail(user.email, 'Critical Alert', message),
      this.sender.sendSms(user.phone, message),
    ]);
  }
}
```

참고: [Interface Segregation Principle](https://en.wikipedia.org/wiki/Interface_segregation_principle)
