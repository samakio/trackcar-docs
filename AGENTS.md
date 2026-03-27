# TrackCar 프로젝트 Agent 가이드라인

이 파일은 TrackCar 차량 운행기록 관리 플랫폼의 **개발 및 유지보수**를 위한 AI 에이전트 가이드라인입니다.

---

## 1. 프로젝트 개요

| 항목 | 내용 |
|------|------|
| **프로젝트** | TrackCar - DTG(Digital Tachograph) 데이터 수집 및 관리 플랫폼 |
| **도메인** | 차량 운행기록 관리, eTAS(한국교통안전공단) 전송 |
| **AWS 계정** | 523911787320 |
| **AWS 리전** | ap-northeast-2 (서울) |
| **GitHub** | https://github.com/samakio/trackcar-aws-infra, https://github.com/samakio/trackcar-aws-functions |

---

## 2. 핵심 설계 원칙

### 2.1 데이터 흐름

```
DTG Simulator ──MQTT──▶ AWS IoT Core ──▶ SQS (raw-ingest)
                                                        │
                                                        ▼
                                              Lambda (ingest-auth-validate)
                                                        │
                               ┌─────────────────────────┼─────────────────────────┐
                               │                         │                         │
                               ▼                         ▼                         ▼
                        S3 (raw)              SQS (clean-queue)              DLQ
                                                         │
                                                         ▼
                                               Lambda (telemetry-process)
                                                         │
                               ┌─────────────────────────┼─────────────────────────┐
                               │                         │                         │
                               ▼                         ▼                         
                        DynamoDB                Aurora (trip_meta)              
```

### 2.2 Polyglot Persistence

| 계층 | 저장소 | 용도 |
|------|--------|------|
| **Hot** | DynamoDB | 실시간 대시보드, 중복 방지 |
| **Warm** | Aurora PostgreSQL | 운행 이력, 메타데이터, 이벤트 로그 |
| **Cold** | S3 (Parquet) | 대용량 원본 보관, eTAS 전송용 |

### 2.3 데이터 보관

| 데이터 타입 | 보관 기간 | 삭제 |
|---------|----------|------|
| S3 Raw Data | 3년 (1095일) | PurgeOldDataHandler |
| Aurora trip_meta, alert_event | 3년 | PurgeOldDataHandler |
| DynamoDB | 실시간 | TTL 또는 직접 삭제 |

### 2.4 eTAS 전송 파이프라인

```
EventBridge (01:00 AM) → etas-fetch-targets → S3 Parquet → [별표 2] TXT 변환 → eTAS 서버 전송
```

### 2.5 디바이스 인증

- **접속**: MQTT over TLS (mTLS)
- **인증서**: X.509 클라이언트 인증서 (Vendor CA 기반)
- **활성화**: Lambda 검증 후 수동 활성화 (자동 활성화 금지)

---

## 3. Lambda 함수

| 함수명 | Handler | 런타임 | 트리거 | 목적 |
|--------|---------|--------|--------|------|
| `ingest-auth-validate` | `IngestAuthValidateHandler` | java25 | SQS (raw-ingest) | 인증/검증, 중복 방지 |
| `telemetry-process` | `TelemetryProcessHandler` | java25 | SQS (clean-queue) | DynamoDB + Aurora 저장 |
| `etas-fetch-targets` | `FetchTargetsHandler` | java25 | EventBridge (일배치) | eTAS 전송 대상 조회 |
| `etas-formatter` | `FormatterHandler` | java25 | SQS (etas-processing) | 파일 생성, S3 업로드 |
| `etas-transmitter` | `TransmitterHandler` | java25 | SQS (etas-transmit) | eTAS API 전송 |
| `api-adapter-client` | `ApiAdapterHandler` | java25 | API Gateway | REST API 요청 처리 |
| `purge-old-data` | `PurgeOldDataHandler` | java25 | EventBridge (02:00 KST) | 3년 된 데이터 자동 삭제 |

### 3.1 API Handler

| 함수명 | Handler | 런타임 | 트리거 | 목적 |
|--------|---------|--------|--------|------|
| `api-adapter-client` | `ApiAdapterHandler` → `MobileHandler` | java25 | API Gateway | Mobile API 요청 처리 |

### 3.2 공통 유틸리티

| 클래스 | 용도 |
|--------|------|
| `CognitoAuthValidator` | JWT 토큰 검증 (JWKS 캐싱) |
| `CognitoUserManager` | Cognito Admin API (사용자 초대/관리) |
| `MobileRequestContext` | Mobile API 요청 context |

---

## 4. 기술 스택

### Backend

| Component | Technology |
|-----------|-------------|
| Language | Java 25 |
| Framework | **Lambda RequestHandler 패턴** (Spring Boot 미사용) |
| AWS SDK | AWS SDK v2 |
| Database | Aurora PostgreSQL (Serverless v2, Data API), DynamoDB |
| Queue | SQS |
| IoT | AWS IoT Core (MQTT5) |
| Auth | AWS Cognito |

> **참고**: 모든 Lambda 함수는 `RequestHandler` 구현체로 작성되며, Aurora 연동은 AWS RDS Data API를 사용합니다.

### Frontend

| 앱 | 플랫폼 | 기술 | 개발 환경 |
|---|--------|------|----------|
| **Installer Admin Web** | Website | React, shadcn/ui, Tailwind CSS | Replit |
| **Mobile App** | iOS, Android | React Native (또는 Flutter) | Expo / Xcode / Android Studio |

---

## 5. Handler 패턴 아키텍처

### 5.1 Handler 클래스 구조

모든 Lambda 함수는 `RequestHandler` 인터페이스를 구현합니다:

```java
public class FetchTargetsHandler implements RequestHandler<EventBridgeEvent, String> {
    private final AuroraRepository auroraRepository;
    
    public FetchTargetsHandler() {
        this.auroraRepository = new AuroraRepository();
    }
    
    @Override
    public String handleRequest(EventBridgeEvent event, Context context) {
        // 비즈니스 로직
        return "SUCCESS";
    }
}
```

### 5.2 주요 Repository

| Repository | 용도 | 구현 방식 |
|------------|------|----------|
| `AuroraRepository` | Aurora PostgreSQL Data API | AWS RDS Data API + Client Library |
| `DynamoDbRepository` | DynamoDB CRUD + 실시간 위치 조회 | AWS SDK v2 |

**AuroraRepository 헬퍼 메서드:**
```java
// Named parameter 바인딩
executeQueryNew(sql, "paramName", paramValue);
executeUpdateNew(sql, "paramName", paramValue);
executeCountNew(sql, "paramName", paramValue);
```

### 5.3 환경변수 참조

Lambda 함수에서 환경변수는 직접 참조합니다 (`System.getenv()`):

```java
// Aurora 연결
String clusterArn = System.getenv("AURORA_CLUSTER_ARN");
String secretArn = System.getenv("AURORA_SECRET_ARN");

// DynamoDB 연결
String tableName = System.getenv("VEHICLE_STATE_TABLE_NAME");
```

---

## 6. 도메인 용어

| 용어 | 설명 |
|------|------|
| **DTG** | Digital Tachograph, 디지털 운행기록장치 |
| **eTAS** | 한국교통안전공단 차량용 단말기 정보포털 |
| **Telemetry** | DTG에서 수집되는 위치/속도/상태 데이터 |
| **Owner** | 화물/여객 운송사업자 (PERSONAL/BUSINESS) |
| **Quick Mapping** | DTG-차량-기사 묶음 매핑 |

---

## 7. 개발 가이드

### Terraform 배포

```bash
cd aws-infra
terraform init && terraform plan -out=tfplan && terraform apply tfplan
```

### Lambda 빌드

```bash
cd aws-functions && ./gradlew build
# ZIP 파일: build/dist/
```

### DTG Simulator 실행

```bash
cd dtg-device-simulator && SPRING_PROFILES_ACTIVE=iot ./gradlew bootRun
```

### eTAS Mock Server 실행

```bash
cd etas-mock-server && ./gradlew bootRun
```

### Aurora 접속 (Data API)

```bash
aws rds-data execute-statement \
  --resource-arn "arn:aws:rds:ap-northeast-2:523911787320:cluster:trackcar-dev-aurora" \
  --secret-arn "arn:aws:secretsmanager:ap-northeast-2:523911787320:secret:trackcar/dev/aurora/master-password" \
  --database "trackcar" --sql "SELECT * FROM trip_meta LIMIT 10;"
```

---

## 8. 문서 참조

| 카테고리 | 문서 |
|----------|------|
| **아키텍처** | `docs/BackEnd/TrackCar 플랫폼 아키텍처 상세 설계.md` |
| **Lambda 구현** | `docs/BackEnd/TrackCar Lambda 구현 설계서 (Java).md` |
| **DynamoDB** | `docs/BackEnd/TrackCar DynamoDB 물리 스키마 명세서.md` |
| **S3** | `docs/BackEnd/TrackCar S3 데이터 카탈로그 명세서.md` |
| **데이터 보관** | `docs/BackEnd/TrackCar 데이터 보관파기 정책.md` |
| **Mobile API** | `docs/FrontEnd/TrackCar Mobile App API Specification.md` |
| **Admin API** | `docs/FrontEnd/TrackCar Installer Admin Web API Specification.md` |
| **법규** | `docs/law/` |

---

## 9. Git 커밋 규칙

**형식:** `[타입] 설명`

| 태그 | 의미 | 예시 |
|------|------|------|
| `[추가]` | 새 기능 | `[추가] AWS IoT MQTT5 클라이언트` |
| `[수정]` | 버그 수정/개선 | `[수정] QOS import 경로 오류` |
| `[문서]` | 문서 추가/수정 | `[문서] API 스펙 업데이트` |
| `[리팩토링]` | 코드 구조 변경 | `[리팩토링] Repository 패턴 적용` |
| `[설정]` | 빌드/환경 설정 | `[설정] .gitignore 추가` |
| `[삭제]` | 파일/코드 삭제 | `[삭제] 미사용 의존성` |

---

## 10. 보안 원칙

| 구분 | 방식 |
|------|------|
| 디바이스 인증 | X.509 클라이언트 인증서 (mTLS) |
| 사용자 인증 | AWS Cognito JWT |
| 비밀 관리 | AWS Secrets Manager |
| PII 보호 | 암호화 + 로그 마스킹 |

---

*Last updated: 2026-03-28*
