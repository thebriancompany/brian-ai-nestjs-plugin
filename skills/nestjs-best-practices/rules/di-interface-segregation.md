---
title: 인터페이스 분리 원칙을 적용한다
impact: HIGH
impactDescription: "사용하지 않는 계약에 대한 의존성을 줄임"
tags: dependency-injection, interfaces, solid, isp
---

## 인터페이스 분리 원칙을 적용한다

소비자가 필요하지 않은 계약에 의존하도록 강요하지 않는다. 서로 다른 사용처나 변경 이유가 있는 인터페이스를 역할별로 나누되, 메서드마다 별도의 인터페이스를 만들 필요는 없다. 타입을 좁히는 것과 런타임 인스턴스를 나누는 것은 별개의 선택이다.

**잘못된 예 (이메일 소비자가 모든 채널의 계약에 의존함):**

```typescript
export const NOTIFICATIONS = Symbol('NOTIFICATIONS');
interface Notifications {
  sendEmail(to: string, subject: string, body: string): Promise<void>;
  sendSms(phone: string, message: string): Promise<void>;
  getDeliveryStatus(id: string): Promise<string>;
}

@Injectable()
export class ReceiptsService {
  constructor(@Inject(NOTIFICATIONS) private readonly notifications: Notifications) {}

  send(to: string, receipt: string): Promise<void> {
    return this.notifications.sendEmail(to, 'Receipt', receipt);
  }
}

// ReceiptsService는 이메일만 쓰지만 대역에도 나머지 계약이 필요하다.
const fakeNotifications: Notifications = {
  sendEmail: jest.fn(),
  sendSms: jest.fn(),
  getDeliveryStatus: jest.fn(),
};
```

**올바른 예 (소비자가 필요한 계약만 주입받음):**

```typescript
// notification-ports.ts — 데코레이터 평가 전에 정의한 같은 토큰을 import한다.
export const EMAIL_SENDER = Symbol('EMAIL_SENDER');
export const SMS_SENDER = Symbol('SMS_SENDER');

export interface EmailSender {
  sendEmail(to: string, subject: string, body: string): Promise<void>;
}
export interface SmsSender {
  sendSms(phone: string, message: string): Promise<void>;
}

@Injectable()
export class ReceiptsService {
  constructor(@Inject(EMAIL_SENDER) private readonly email: EmailSender) {}

  send(to: string, receipt: string): Promise<void> {
    return this.email.sendEmail(to, 'Receipt', receipt);
  }
}

const fakeEmail: EmailSender = { sendEmail: jest.fn() };

// 기존 NotificationService가 EmailSender와 SmsSender를 모두 구현한다고 가정한다.
// 외부 전송의 구현·설정은 해당 서비스가 소유한다.
@Module({
  providers: [
    NotificationService,
    { provide: EMAIL_SENDER, useExisting: NotificationService },
    { provide: SMS_SENDER, useExisting: NotificationService },
  ],
  exports: [EMAIL_SENDER, SMS_SENDER],
})
export class NotificationModule {}

@Module({
  imports: [NotificationModule],
  providers: [ReceiptsService],
})
export class ReceiptsModule {}
```

`useExisting`은 등록된 기본 스코프 `NotificationService` 인스턴스를 두 계약으로 노출한다. 실제 전송 구현도 분리할 이유가 있다면 각 토큰에 별도의 `useClass`를 등록한다. 타입 선언만 좁힌다고 Nest가 인터페이스를 알아서 주입하지는 않으므로 토큰·프로바이더·모듈 경계를 함께 맞춘다.

여러 채널을 쓰는 소비자는 `EMAIL_SENDER`와 `SMS_SENDER`를 각각 주입받을 수 있다. 별도 복합 계약이 실제로 필요한 경우에만 교차 타입과 그 타입을 만족하는 구현의 토큰을 등록한다. 교차 타입 자체가 런타임 토큰은 아니다.

참고: [NestJS Custom Providers](https://docs.nestjs.com/fundamentals/custom-providers)
