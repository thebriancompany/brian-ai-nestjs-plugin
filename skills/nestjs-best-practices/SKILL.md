---
name: nestjs-best-practices
description: 운영 환경에 적합한 애플리케이션을 만들기 위한 NestJS 모범 사례와 아키텍처 패턴. NestJS 코드를 작성·리뷰·리팩터링할 때 사용하며, 모듈, 의존성 주입, 보안, 성능에 적절한 패턴을 적용하도록 안내한다.
metadata:
  version: "1.2.1-ko.1"
---

# NestJS 모범 사례

NestJS 애플리케이션을 위한 종합 모범 사례 가이드다. 자동 리팩터링과 코드 생성을 돕도록 10개 범주의 규칙 40개를 영향도에 따라 우선순위별로 정리했다.

## 적용 시점

다음 작업에서 이 지침을 참고한다.

- 새로운 NestJS 모듈, 컨트롤러, 서비스 작성
- 인증과 인가 구현
- 아키텍처와 보안 문제를 찾기 위한 코드 리뷰
- 기존 NestJS 코드베이스 리팩터링
- 성능이나 데이터베이스 쿼리 최적화
- 마이크로서비스 아키텍처 구축

## 적용 절차와 판단 기준

1. 먼저 대상 프로젝트의 NestJS·ORM·주변 패키지 버전, 실행 환경, 기존 규칙과 테스트를 확인한다. 이 스킬의 수정 예제는 NestJS 11, TypeORM 0.3, BullMQ 5, Throttler 6을 기준으로 한다. 다른 버전에는 설치된 패키지와 공식 문서에 맞춰 적용한다.
2. 현재 작업에 관련된 규칙 파일만 읽고 실제 장애 조건을 확인한다. 예제는 주제별 발췌이며 도메인 타입·프로바이더·모듈 등록을 프로젝트에 맞게 완성해야 한다.
3. 보안, 데이터 정합성, 실행 오류부터 해결한다. 표의 영향도는 참고 분류이며 모든 프로젝트에서 같은 우선순위를 뜻하지 않는다. 선호하는 패턴과 버그를 구분하고 근거 없이 필수 위반으로 판정하지 않는다.
4. 작은 CRUD를 위해 별도 리포지토리·이벤트·큐를 강제하거나 기존 스택을 교체하지 않는다. 요청 범위에 필요한 최소 변경을 선택한다.
5. 변경한 동작의 정상·실패 경로를 검증한다. 코드 리뷰에는 발생 조건, 영향, 파일 위치와 재현 근거를 포함한다. 확인하지 못한 의존성이나 운영 조건은 명시한다.

## 우선순위별 규칙 범주

| 우선순위 | 범주 | 영향도 | 접두사 |
|----------|----------|--------|--------|
| 1 | 아키텍처 | CRITICAL | `arch-` |
| 2 | 의존성 주입 | CRITICAL | `di-` |
| 3 | 오류 처리 | HIGH | `error-` |
| 4 | 보안 | HIGH | `security-` |
| 5 | 성능 | HIGH | `perf-` |
| 6 | 테스트 | MEDIUM-HIGH | `test-` |
| 7 | 데이터베이스와 ORM | MEDIUM-HIGH | `db-` |
| 8 | API 설계 | MEDIUM | `api-` |
| 9 | 마이크로서비스 | MEDIUM | `micro-` |
| 10 | DevOps와 배포 | LOW-MEDIUM | `devops-` |

## 빠른 참조

### 1. 아키텍처 (CRITICAL)

- `arch-avoid-circular-deps` - 모듈의 순환 의존성을 피한다
- `arch-feature-modules` - 기술 계층이 아닌 기능별로 구성한다
- `arch-module-sharing` - 모듈을 올바르게 내보내고 가져오며 중복 프로바이더를 피한다
- `arch-single-responsibility` - 거대 서비스 대신 책임에 집중하는 서비스를 만든다
- `arch-use-repository-pattern` - 테스트를 쉽게 하도록 데이터베이스 로직을 추상화한다
- `arch-use-events` - 이벤트 기반 아키텍처로 결합도를 낮춘다

### 2. 의존성 주입 (CRITICAL)

- `di-avoid-service-locator` - 서비스 로케이터 안티패턴을 피한다
- `di-interface-segregation` - 인터페이스 분리 원칙(ISP)을 적용한다
- `di-liskov-substitution` - 리스코프 치환 원칙(LSP)을 지킨다
- `di-prefer-constructor-injection` - 프로퍼티 주입보다 생성자 주입을 우선한다
- `di-scope-awareness` - 싱글턴·요청·일시적 스코프를 이해한다
- `di-use-interfaces-tokens` - 인터페이스에 주입 토큰을 사용한다

### 3. 오류 처리 (HIGH)

- `error-use-exception-filters` - 예외 처리를 중앙화한다
- `error-throw-http-exceptions` - NestJS HTTP 예외를 사용한다
- `error-handle-async-errors` - 비동기 오류를 올바르게 처리한다

### 4. 보안 (HIGH)

- `security-auth-jwt` - 안전한 JWT 인증을 구현한다
- `security-validate-all-input` - class-validator로 검증한다
- `security-use-guards` - 인증과 인가에 가드를 사용한다
- `security-sanitize-output` - XSS 공격을 방지한다
- `security-rate-limiting` - 요청 횟수 제한을 구현한다

### 5. 성능 (HIGH)

- `perf-async-hooks` - 비동기 생명주기 훅을 올바르게 사용한다
- `perf-use-caching` - 캐싱 전략을 구현한다
- `perf-optimize-database` - 데이터베이스 쿼리를 최적화한다
- `perf-lazy-loading` - 시작 속도를 높이기 위해 모듈을 지연 로딩한다

### 6. 테스트 (MEDIUM-HIGH)

- `test-use-testing-module` - NestJS 테스트 도구를 사용한다
- `test-e2e-supertest` - Supertest로 E2E 테스트를 수행한다
- `test-mock-external-services` - 외부 의존성을 모의 객체로 대체한다

### 7. 데이터베이스와 ORM (MEDIUM-HIGH)

- `db-use-transactions` - 트랜잭션을 관리한다
- `db-avoid-n-plus-one` - N+1 쿼리 문제를 피한다
- `db-use-migrations` - 스키마 변경에 마이그레이션을 사용한다

### 8. API 설계 (MEDIUM)

- `api-use-dto-serialization` - DTO와 응답 직렬화를 사용한다
- `api-use-interceptors` - 공통 관심사를 처리한다
- `api-versioning` - API 버전 관리 전략을 적용한다
- `api-use-pipes` - 파이프로 입력을 변환한다

### 9. 마이크로서비스 (MEDIUM)

- `micro-use-patterns` - 메시지 패턴과 이벤트 패턴을 사용한다
- `micro-use-health-checks` - 오케스트레이션을 위한 상태 검사를 수행한다
- `micro-use-queues` - 백그라운드 작업을 처리한다

### 10. DevOps와 배포 (LOW-MEDIUM)

- `devops-use-config-module` - 환경 설정을 관리한다
- `devops-use-logging` - 구조화된 로깅을 사용한다
- `devops-graceful-shutdown` - 무중단 배포를 지원한다

## 사용 방법

자세한 설명과 코드 예제는 각 규칙 파일에서 확인한다.

```
rules/arch-avoid-circular-deps.md
rules/security-validate-all-input.md
rules/_sections.md
```

각 규칙 파일에는 다음 내용이 담겨 있다.
- 해당 규칙이 중요한 이유에 대한 간단한 설명
- 잘못된 코드 예제와 설명
- 올바른 코드 예제와 설명
- 추가 맥락과 참고 자료
