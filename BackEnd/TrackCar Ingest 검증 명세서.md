# TrackCar Ingest 검증 명세서

- 버전: v3.0
- 수정일: 2026-03-25
- 기준 문서:
  - `DTG 단말기 ↔ TrackCar IoT Core 연동 요구사항`
  - `TrackCar 디바이스 Telemetry 인터페이스 명세`
  - `TrackCar 플랫폼 아키텍처 상세 설계 v3.0.0`
  - `TrackCar 웹 데이터모델 & ERD`
  - `TrackCar 이벤트 계약서 (내부 공통)`
- 목적: IoT Core 기반 **MQTT 전용 ingest** 경로에서 등록/매핑/권한/멱등 검증을 표준화한다.
- 해석 기준: 본 문서는 **TrackCar 플랫폼 아키텍처 상세 설계 v3.0.0** 기준으로 해석/구현한다.

---

## 1) 범위 및 원칙

1. Ingest 경로는 **MQTT only** (`AWS IoT Core`)로 운영한다.
2. REST ingest 경로는 운영 대상에서 제외한다.
3. IoT Core 인증(mTLS)은 필수이지만, 인증 통과만으로 테넌트/권한을 신뢰하지 않는다.
4. 서버는 매핑 테이블 기준으로 최종 권한/유효성 판단을 수행한다.
5. 실시간 수집 경로는 **AWS IoT Core(MQTT/mTLS) → IoT Rules → SQS(raw-ingest) → ingest-auth-validate → SQS(clean-queue)** 기준을 따른다.

---

## 2) 온보딩/사전 조건

1. 제조사 Vendor CA는 AWS IoT Core에 신뢰 CA로 등록되어 있어야 한다.
2. 디바이스 인증서는 JITR 기준으로 자동 등록될 수 있으나, 초기 상태는 검증 대기(PENDING_ACTIVATION 상당)로 본다.
3. Lambda 검증 전에는 인증서를 자동 활성화하지 않는다.
4. 검증 키는 `deviceId` 우선이며, 인증서 subject/serial은 교차 검증에 사용한다.

## 3) 입력 계약 (MQTT)

- Topic 패턴: `trackcar/{deviceId}/telemetry`
- Payload: 제조사별 원본 payload + 공통 운영 필드
- 공통 필수 필드(권장):
  - `interfaceCode`
  - `interfaceVersion`
  - `batchId` (또는 동등 멱등 키)
  - `sentAt`
- 차량 상태 필드는 `ignition` 을 단일 기준으로 사용한다.
- `engineOn` 은 현재 표준 입력 계약에서 사용하지 않으며, 제조사 내부 상태값이 있더라도 표준 payload에서는 `ignition` 기준으로 정리한다.

참고:
- topic은 `deviceId` 기준으로 단순화하며, tenant 소속 판단은 서버 매핑값을 단일 기준으로 사용한다.

---

## 4) 검증 순서 (고정)

1. **기본 형식 검증**
   - payload 최소 필드/타입 검증
   - point 상태 필드는 `ignition` 단일 기준으로 해석
2. **디바이스 등록/상태 검증**
   - `device` 조회
   - 상태 `ACTIVE` 여부 확인
3. **테넌트-디바이스 매핑 검증**
   - `deviceId` 기준 서버 매핑 tenant와 binding 일치 검증
4. **차량-디바이스 바인딩 검증**
   - `vehicle_device_binding` 존재/활성 확인
5. **인터페이스 버전 검증**
   - payload `interfaceCode/interfaceVersion`
   - `dtg_interface_catalog(interface_code, version)` 활성 상태
   - `vehicle_device_binding.interface_id/interface_version`과 일치 검증
6. **구독 상태 검증**
   - `subscription.status = ACTIVE`만 허용
7. **Outbound 연계 유효성 검증**
   - `outbound_profile` 활성 검증
   - `outbound_trigger_policy` 유효성 검증
8. **멱등/재전송 검증**
   - `deviceId + batchId`(또는 정의된 멱등 키) 기준 중복 처리
9. **통과 시 Canonical 이벤트 발행**
   - `TELEMETRY_CANONICAL_BATCH` (이벤트 계약 v2.0 준수)

---

## 5) 에러 코드 (MQTT 기준)

### 4.1 공개/표준 코드
- `INVALID_PARAM`
- `DEVICE_NOT_REGISTERED`
- `DEVICE_INACTIVE`
- `TENANT_DEVICE_MISMATCH`
- `DEVICE_INTERFACE_MAPPING_NOT_FOUND`
- `INTERFACE_VERSION_MISMATCH`
- `SUBSCRIPTION_NOT_ACTIVE`
- `OUTBOUND_PROFILE_INVALID`
- `REPLAY_DETECTED`

### 4.2 내부 상세 코드(로그)
- `TOPIC_TENANT_MISMATCH`
- `BINDING_NOT_ACTIVE`
- `INTERFACE_NOT_ACTIVE`
- `SUBSCRIPTION_SUSPENDED`
- `SUBSCRIPTION_UNPAID`
- `SUBSCRIPTION_CANCELED`
- `IDEMPOTENCY_KEY_ALREADY_USED`

---

## 6) 보안/멱등 정책

1. 인증
- IoT Core mTLS 정책을 기본 인증으로 사용
- 인증서/정책은 환경별 최소권한 원칙 적용

2. 권한 검증
- 토픽 정보는 보조 신호이며, 서버 DB 매핑을 권한 기준으로 사용

3. 멱등
- `idempotency_store` 또는 동등 저장소로 중복 차단
- 멱등키는 `deviceId + batchId` 기준을 우선 적용
- Dedup Guard는 DynamoDB TTL 기반으로 구현 가능해야 한다.
- TTL 정책: 운영 기준 1~7일

4. 데이터/연결 정책
- ingest-auth-validate 계층의 RDB 연결은 **RDS Proxy 경유**를 기본 원칙으로 한다.

---

## 7) 관측/운영 지표

- ingest accepted/rejected count
- reject by stage (형식/디바이스/매핑/인터페이스/구독/멱등)
- reject by `errorCode`
- interface_code/version 별 실패율
- validation latency p95/p99
- tenant 별 ingest volume
- 재시도 횟수 및 지수 백오프 적용 현황
- 전송/연계 로그 적재 여부 (`etas_transmission_log` 포함)

운영 참고:
- 비즈니스 처리는 `telemetry-process`가 **DynamoDB 최신 상태 Upsert + RDS PostgreSQL 이벤트/트립 적재** 수행 기준을 따른다.
- Cold 데이터는 **Lambda 직접 S3 Parquet** 저장 및 장기보관 정책을 따른다.
- eTAS 연계는 **실시간 분리형 심야 배치** 구조를 전제로 하며, 포맷 및 파일명 규격을 준수해야 한다.

---

## 8) 테스트 최소셋

1. 정상 등록 매핑 ACTIVE 구독 통과
2. 미등록/비활성 디바이스 차단
3. device 매핑 tenant 불일치 차단
4. 바인딩 없음/비활성 차단
5. interface code/version 불일치 차단
6. 구독 비활성 차단
7. 멱등 중복 차단
8. Canonical 이벤트 출력 스키마(v2.0) 검증
9. RDS Proxy 경유 연결 및 재시도/전송로그 기록 기준 검증

---

## 9) 변경 이력

- v3.0 (2026-03-25)
  - Kinesis Data Streams → SQS 변경 (Free Tier 호환)
  - Kinesis Firehose → Lambda 직접 S3 Parquet 적재로 변경
  - Aurora PostgreSQL → RDS PostgreSQL Single-AZ 변경
  - TrackCar 플랫폼 아키텍처 상세 설계 v3.0.0 기준 적용
- v2.2 (2026-03-18)
  - DTG 표준 상태 필드를 `ignition` 단일 기준으로 명시하고 `engineOn` 비사용 원칙 반영
- v2.1 (2026-03-02)
  - TrackCar 아키텍처 심화 설계 v2.4.0 기준 정합화
  - 실시간 수집 경로, 멱등키 기준, RDS Proxy 경유 원칙 반영
  - eTAS 포맷/전송로그 기준 참조 명시
- v2.0 (2026-03-01)
  - Ingest 경로를 MQTT(IoT Core) 전용으로 확정
  - REST/HMAC/Nonce 검증 항목 제거
  - 인터페이스/바인딩/아웃바운드 프로파일 검증 단계 보강
  - 이벤트 계약 v2.0 메타 정합성 반영
- v1.1
  - MQTT deviceId 고정 토픽/구독 상태 검증 반영
- v1.0
  - 최초 작성

#업무/프로젝트/trackcar/산출물
