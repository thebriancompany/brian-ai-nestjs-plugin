---
title: 백그라운드 작업에 메시지 큐를 사용한다
impact: MEDIUM-HIGH
impactDescription: "안정적인 백그라운드 처리를 지원함"
tags: microservices, queues, bullmq, background-jobs
---

## 백그라운드 작업에 메시지 큐를 사용한다

백그라운드 작업 처리에 `@nestjs/bullmq`를 사용한다. 큐는 오래 걸리는 작업을 HTTP 요청에서 분리하고, 재시도 로직을 적용하며, 여러 워커에 작업을 분산한다. 이메일, 파일 처리, 알림 등 사용자 요청을 지연시키지 않아야 하는 작업에 활용한다.

**잘못된 예 (HTTP 핸들러에서 오래 걸리는 작업을 처리함):**

```typescript
// HTTP 핸들러 안의 장기 실행 작업
@Controller('reports')
export class ReportsController {
  @Post()
  async generate(@Body() dto: GenerateReportDto): Promise<Report> {
    // 요청을 수 분간 지연시킬 수 있음
    const data = await this.fetchLargeDataset(dto);
    const report = await this.processData(data); // 느림!
    await this.sendEmail(dto.email, report); // 실패할 수 있음!
    return report; // 클라이언트에서 시간 초과 발생
  }
}

// 재시도 없이 직접 실행함
@Injectable()
export class EmailService {
  async sendWelcome(email: string): Promise<void> {
    // 실패하면 이메일을 전송하지 못함
    await this.mailer.send({ to: email, template: 'welcome' });
    // 재시도와 추적이 없고 상태를 확인할 수 없음
  }
}

// 예약 작업에 setInterval 사용
setInterval(async () => {
  await cleanupOldRecords();
}, 60000); // 실패 시 거부된 Promise가 처리되지 않음
```

**올바른 예 (BullMQ의 WorkerHost와 이벤트 API를 사용함):**

`@nestjs/bullmq` 11과 BullMQ 5.16 이상을 기준으로 한다. Bull 패키지의 `@Process`, `@OnQueueCompleted`와 혼용하지 않는다.

```typescript
import { BullModule, InjectQueue, Processor, WorkerHost, OnWorkerEvent } from '@nestjs/bullmq';
import { Job, Queue } from 'bullmq';

@Injectable()
export class ReportsService {
  constructor(@InjectQueue('reports') private reportsQueue: Queue) {}

  async requestReport(dto: GenerateReportDto, ownerId: string, requestId: string): Promise<{ jobId: string }> {
    const scheduledAt = dto.scheduledFor ? Date.parse(dto.scheduledFor) : Date.now();
    if (!Number.isFinite(scheduledAt)) throw new BadRequestException('Invalid schedule');
    const job = await this.reportsQueue.add('generate', { parameters: dto, ownerId, requestId }, {
      priority: 10,
      delay: Math.max(0, scheduledAt - Date.now()),
    });
    if (!job.id) throw new Error('Missing job ID');
    return { jobId: job.id };
  }

  async getJobStatus(jobId: string, ownerId: string) {
    // ownerId는 인증된 주체에서 전달하며 입력 DTO에서 받지 않는다.
    const job = await this.reportsQueue.getJob(jobId);
    if (!job || job.data.ownerId !== ownerId) throw new NotFoundException('Job not found');
    return { status: await job.getState(), progress: job.progress, result: job.returnvalue };
  }
}

// requestId는 재시도에도 동일한 검증된 업무 요청 ID다. ownerId는 인증 정보에서 온다.
interface ReportJobData {
  parameters: GenerateReportDto;
  ownerId: string;
  requestId: string;
}

@Processor('reports')
export class ReportsProcessor extends WorkerHost {
  constructor(private readonly generator: ReportGenerator) { super(); }

  async process(job: Job<ReportJobData>): Promise<Report> {
    switch (job.name) {
      case 'generate':
        // generator는 (ownerId, requestId)를 고유 키로 사용하며 재시도 시 기존 결과를 반환한다.
        // 같은 키에 다른 parameters가 오면 충돌로 거부한다.
        await job.updateProgress(10);
        const report = await this.generator.generateAndSave(job.data);
        await job.updateProgress(100);
        return report;
      default:
        throw new Error(`Unknown report job: ${job.name}`);
    }
  }

  @OnWorkerEvent('failed')
  onFailed(job: Job | undefined, error: Error): void {
    // failed 이벤트에서는 job이 없을 수 있다. 민감한 payload는 기록하지 않는다.
    Logger.error(`Report job ${job?.id ?? 'unknown'} failed`, error.stack);
  }
}

@Processor('email')
export class EmailProcessor extends WorkerHost {
  constructor(private readonly mailer: MailerService) { super(); }

  async process(job: Job<SendEmailDto>): Promise<void> {
    if (job.name !== 'send') throw new Error(`Unknown email job: ${job.name}`);
    // 업무 이벤트 ID로 중복 발송을 막는 서비스. 실패는 재시도를 위해 전파한다.
    await this.mailer.sendOnce(job.data);
  }
}

@Injectable()
export class ScheduledJobsService implements OnModuleInit {
  constructor(@InjectQueue('maintenance') private queue: Queue) {}

  async onModuleInit(): Promise<void> {
    await this.queue.upsertJobScheduler('daily-cleanup',
      { pattern: '0 0 * * *', tz: 'UTC' },
      { name: 'cleanup', data: {} });
  }
}

@Processor('maintenance')
export class MaintenanceProcessor extends WorkerHost {
  constructor(private readonly maintenance: MaintenanceService) { super(); }
  async process(job: Job): Promise<void> {
    if (job.name !== 'cleanup') throw new Error(`Unknown maintenance job: ${job.name}`);
    await this.maintenance.cleanupExpiredRecords();
  }
}

@Module({
  imports: [
    // ReportGenerator, MailerService, MaintenanceService를 내보내는 앱 모듈
    ReportModule, MailModule, MaintenanceModule,
    BullModule.forRoot({
      connection: { host: 'localhost', port: 6379 },
      defaultJobOptions: {
        removeOnComplete: 1000,
        removeOnFail: 5000,
        attempts: 3,
        backoff: { type: 'exponential', delay: 1000 },
      },
    }),
    BullModule.registerQueue({ name: 'email' }, { name: 'reports' }, { name: 'maintenance' }),
  ],
  providers: [ReportsService, ReportsProcessor, EmailProcessor,
    MaintenanceProcessor, ScheduledJobsService],
  exports: [ReportsService, BullModule],
})
export class QueueModule {}
```

지연 시간은 실행 가능한 가장 이른 시각이며 정확한 실행 시각을 보장하지 않는다. 우선순위·예약 가능 기간·요청 빈도는 API에서 검증하며 클라이언트가 무제한 높은 우선순위를 지정하게 하지 않는다.

워커 작업은 중복 실행될 수 있다. `attempts` 설정만으로 멱등성이 보장되지는 않는다. 예제의 `generateAndSave`와 `sendOnce`는 앱에서 구현할 업무 계약이며, DB의 고유 키나 발송 제공자의 멱등 키를 이용해야 한다. `jobId` 중복 방지도 완료 작업이 제거된 뒤에는 영구적인 중복 방지를 보장하지 않는다. DB 저장과 큐 발행을 함께 보장해야 한다면 트랜잭셔널 아웃박스를 사용한다.

상태 조회는 작업 ID만으로 허용하지 말고 저장한 소유자와 현재 사용자를 대조한다. 여러 인스턴스의 예약 등록은 안정적인 scheduler ID를 공유한다. 큐 관리 화면을 추가한다면 라우트를 공개하기 전에 관리자 인증·인가를 적용한다. 큐 지연, 실패, 재시도 횟수와 워커 상태를 모니터링한다.

참고: [NestJS Queues](https://docs.nestjs.com/techniques/queues), [BullMQ Job Schedulers](https://docs.bullmq.io/guide/job-schedulers)
