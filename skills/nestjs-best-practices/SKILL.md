---
name: nestjs-best-practices
description: NestJS 애플리케이션을 작성·리뷰·리팩터링할 때 사용하는 한국어 가이드. 모듈과 DI, 인증·인가, 데이터 정합성, 비동기 처리, 테스트와 운영 예제를 프로젝트의 버전에 맞춰 적용한다.
metadata:
  version: "1.2.1-ko.2"
---

# NestJS 모범 사례

10개 범주, 40개 규칙을 제공한다. 프로젝트의 기존 구조와 실제 실패 조건에 맞는 규칙만 선택해 사용한다.

## 적용 절차

1. 프로젝트의 `package.json`·잠금 파일, NestJS 버전, HTTP 어댑터, ORM·DB 종류, 테스트와 기존 규칙을 먼저 확인한다. 기본 문서는 HTTP 예제이며 Express 전용 코드는 Fastify·GraphQL·RPC에 그대로 적용하지 않는다.
2. 아래에서 관련 규칙만 읽는다. 보안 경계·데이터 정합성·실행 실패를 먼저 확인한 뒤 유지보수와 측정된 성능 문제를 다룬다. 파일의 `impact`는 참고값이며 폴더 순서나 패턴 선호만으로 리뷰 심각도를 정하지 않는다.
3. 예제는 주제별 발췌다. 잘못된 예를 복사하지 말고, 대안 예제의 같은 이름 클래스·모듈을 한 파일에 합치지 않는다. 도메인 타입·주입 토큰·provider·import를 프로젝트에 맞게 연결하되 예제의 핵심 검증이나 실패 처리를 생략하지 않는다.
4. 작은 CRUD에 별도 리포지토리·이벤트·큐·추상화를 강제하지 않는다. ORM·로거·인증 방식도 필요 없이 교체하지 않는다. 전역 가드·파이프·인터셉터·종료 신호는 기존 등록 지점과 통합해 중복 실행을 피한다.
5. 변경한 정상·실패 경로를 검증한다. 인증 변경은 현재 사용자 상태와 권한, DB 변경은 해당 DB의 고유 제약·트랜잭션, 캐시는 사용자 격리와 장애·오래된 값, 운영 코드는 초기화·종료 실패까지 확인한다. 주입한 모의 객체나 타입 검사만으로 실제 DB·네트워크 동작을 검증했다고 주장하지 않는다.
6. 리뷰 결과에는 발생 조건, 영향, 파일 위치와 재현 근거를 쓴다. 설계 대안과 버그를 구분하고 확인하지 못한 통합 환경은 명시한다.

## 버전과 예제 범위

기준은 NestJS 11, TypeORM 0.3, BullMQ 5.16 이상, `@nestjs/throttler` 6, `@nestjs/terminus` 11이다. 캐시는 `@nestjs/cache-manager` 3과 `cache-manager` 7/Keyv를 기준으로 하며 Redis 어댑터·클라이언트 버전은 해당 규칙에 명시한다. 최신 NestJS 기본 문서에는 다른 주 버전의 API가 있을 수 있으므로 [NestJS 11 문서](https://docs.nestjs.com/v11/)와 설치된 패키지 타입·소스를 함께 확인한다. 기준과 다른 버전을 자동 업그레이드하지 않는다.

일반적인 NestJS import·도메인 모델은 일부 생략한다. `Request`·`Response`를 사용하는 Express 예제는 DOM 타입과 혼동하지 않는다. 돈의 단위, 토큰 무효화 정책, 테넌트 경계, 멱등 키 같은 업무 계약은 적용 전에 정한다. 예제에 설명된 외부 서비스 계약은 실제 구현이 제공된다는 뜻이 아니다.

## 규칙 탐색

범주는 탐색용이며 우선순위 순서가 아니다. 범주 설명은 [범주 안내](rules/_sections.md)를 참고한다.

### 아키텍처

- [순환 의존성을 피한다](rules/arch-avoid-circular-deps.md)
- [기능별 모듈로 구성한다](rules/arch-feature-modules.md)
- [올바른 모듈 공유 패턴을 사용한다](rules/arch-module-sharing.md)
- [서비스에 단일 책임을 부여한다](rules/arch-single-responsibility.md)
- [이벤트 기반 아키텍처로 결합도를 낮춘다](rules/arch-use-events.md)
- [데이터 접근에 리포지토리 패턴을 사용한다](rules/arch-use-repository-pattern.md)

### 의존성 주입

- [서비스 로케이터 안티패턴을 피한다](rules/di-avoid-service-locator.md)
- [인터페이스 분리 원칙을 적용한다](rules/di-interface-segregation.md)
- [리스코프 치환 원칙을 지킨다](rules/di-liskov-substitution.md)
- [생성자 주입을 우선한다](rules/di-prefer-constructor-injection.md)
- [프로바이더 스코프를 이해한다](rules/di-scope-awareness.md)
- [인터페이스에 주입 토큰을 사용한다](rules/di-use-interfaces-tokens.md)

### 오류 처리

- [비동기 오류를 올바르게 처리한다](rules/error-handle-async-errors.md)
- [HTTP 경계에 맞는 예외를 던진다](rules/error-throw-http-exceptions.md)
- [예외 필터로 오류를 처리한다](rules/error-use-exception-filters.md)

### 보안

- [안전한 JWT 인증을 구현한다](rules/security-auth-jwt.md)
- [요청 횟수 제한을 구현한다](rules/security-rate-limiting.md)
- [출력 문맥에 맞게 XSS를 방지한다](rules/security-sanitize-output.md)
- [인증과 인가에 가드를 사용한다](rules/security-use-guards.md)
- [DTO와 파이프로 모든 입력을 검증한다](rules/security-validate-all-input.md)

### 성능

- [비동기 생명주기 훅을 올바르게 사용한다](rules/perf-async-hooks.md)
- [선택적 프로바이더에 지연 로딩을 사용한다](rules/perf-lazy-loading.md)
- [데이터베이스 쿼리를 최적화한다](rules/perf-optimize-database.md)
- [캐싱을 전략적으로 사용한다](rules/perf-use-caching.md)

### 테스트

- [E2E 테스트에 Supertest를 사용한다](rules/test-e2e-supertest.md)
- [단위 테스트에서 외부 서비스를 모의 객체로 대체한다](rules/test-mock-external-services.md)
- [DI 구성을 검증할 때 TestingModule을 사용한다](rules/test-use-testing-module.md)

### 데이터베이스와 ORM

- [N+1 쿼리 문제를 피한다](rules/db-avoid-n-plus-one.md)
- [데이터베이스 마이그레이션을 사용한다](rules/db-use-migrations.md)
- [여러 단계의 작업에 트랜잭션을 사용한다](rules/db-use-transactions.md)

### API 설계

- [API 응답에 DTO와 직렬화를 사용한다](rules/api-use-dto-serialization.md)
- [공통 관심사에 인터셉터를 사용한다](rules/api-use-interceptors.md)
- [입력 변환에 파이프를 사용한다](rules/api-use-pipes.md)
- [호환성을 깨뜨리는 변경에는 API 버전을 사용한다](rules/api-versioning.md)

### 마이크로서비스

- [마이크로서비스에 상태 검사를 구현한다](rules/micro-use-health-checks.md)
- [메시지 패턴과 이벤트 패턴을 올바르게 사용한다](rules/micro-use-patterns.md)
- [백그라운드 작업에 메시지 큐를 사용한다](rules/micro-use-queues.md)

### DevOps와 배포

- [안전한 종료를 구현한다](rules/devops-graceful-shutdown.md)
- [환경 설정에 ConfigModule을 사용한다](rules/devops-use-config-module.md)
- [구조화된 로깅을 사용한다](rules/devops-use-logging.md)

## 예제를 함께 적용할 때

- [JWT 전략](rules/security-auth-jwt.md)과 [전역 가드](rules/security-use-guards.md)를 함께 사용하고, [요청 제한](rules/security-rate-limiting.md)을 인증 전후 중 적절한 위치에 둔다. [CLS 사용자 문맥](rules/devops-use-logging.md)은 인증 성공 이후 설정한다.
- [입력 파이프](rules/api-use-pipes.md)와 [입력 DTO](rules/security-validate-all-input.md)를 같은 전역 설정으로 검증한다. [응답 DTO](rules/api-use-dto-serialization.md)는 입력 권한을 대신하지 않는다.
- [준비 상태 검사](rules/micro-use-health-checks.md)와 [종료 처리](rules/devops-graceful-shutdown.md)는 같은 종료 상태 인스턴스를 공유한다. [시간 초과](rules/api-use-interceptors.md)가 실제 작업을 취소하는지도 확인한다.
- [트랜잭션](rules/db-use-transactions.md)과 [이벤트](rules/arch-use-events.md)·[큐](rules/micro-use-queues.md)를 함께 사용하면 커밋과 전달 사이의 실패를 검토한다. [E2E](rules/test-e2e-supertest.md)는 실제 앱 설정을 공유하고 외부 자원은 격리한다.

규칙을 추가·수정할 때는 [작성 템플릿](rules/_template.md)의 검증 기준을 적용한다.
