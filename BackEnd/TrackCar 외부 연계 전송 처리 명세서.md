# TrackCar 외부 연계 전송 처리 명세서
- 작성일: 2026-02-13
- 수정일: 2026-03-25
- 버전: v2.0
- 기반: TrackCar 플랫폼 아키텍처 상세 설계 v3.0.0
- 기준 문서:
  - `TrackCar Web PRD`
  - `TrackCar Architecture DeepDive`
  - `TrackCar 이벤트 계약서 (내부 공통)`
  - `차량별 1일 운행기록 ETAS 연계 설계서 (초안)`
  - `[별표 2] 운행기록의 배열순서`
  - `[별표 3] 운행기록의 파일명`

---

## 1) 목적

본 문서는 TrackCar에서 외부 연계 대상 시스템으로 데이터를 전송할 때의 공통 처리 원칙을 정의한다.
현재 우선 적용 대상은 **eTAS 연계**이며, 특히 **차량별 1일 운행기록의 생성·보강·검증·전송·재시도·이력관리**를 구체적으로 다룬다.

즉, 본 문서는 단순한 범용 outbound 엔진 설명이 아니라,
**TrackCar 내부 데이터와 Admin Web 기준정보를 결합하여 eTAS 제출 단위로 안전하게 전송하는 처리 명세**를 목적으로 한다.

---

## 2) 적용 범위

### 2.1 현재 범위(우선)
- 외부 연계 대상: **eTAS**
- 전송 단위: 차량별 1일 운행기록
- 기본 트리거: `DAILY_CLOSE`
- 보조 트리거: 필요 시 `TRIP_ENDED` 기반 사전 생성/검증
- 결과 관리: 성공/실패/재시도/DLQ/감사로그

### 2.2 향후 확장 범위
- 향후 다른 파트너/관제/정산 시스템 연계에도 동일한 dispatch 구조를 재사용할 수 있다.
- 단, 본 문서의 상세 규칙(필드 보강, 파일명 규칙, 법규 검증)은 현재 **eTAS 우선 기준**으로 정의한다.

---

## 3) eTAS 연계 처리 개념

### 3.1 처리 흐름 요약
1. TrackCar가 telemetry/운행 데이터를 수집한다.
2. Admin Web에서 사전 등록한 차량/사업자/운전자 기준정보를 조회한다.
3. 서버가 일 단위 운행기록을 집계/보강한다.
4. eTAS 제출 규격에 맞게 검증한다.
5. eTAS 인터페이스로 전송한다.
6. 전송 이력과 응답을 저장한다.

### 3.2 핵심 원칙
- DTG 단말 payload와 eTAS 제출 포맷은 동일하지 않다.
- eTAS 제출에 필요한 차량/사업자/운전자 메타는 **Admin Web 기준정보**로 보강한다.
- eTAS 전송 직전 반드시 법규 기반 형식/자릿수/필수값 검증을 수행한다.
- 기본 전송 기준은 **차량별/일자별 1건**이다.

---

## 4) 입력 트리거

### 4.1 기본 트리거
1. `DAILY_CLOSE` 스케줄 이벤트 (EventBridge Scheduler)
   - 매일 마감 시점에 차량별 일일 운행기록 생성 및 eTAS 전송 수행

### 4.2 보조 트리거
2. `TRIP_ENDED` 이벤트 (SQS)
   - 운행 종료 시점에 중간 집계/사전 검증/캐시 생성 가능
   - 단, eTAS의 공식 제출 단위는 기본적으로 `DAILY_CLOSE`를 우선 적용

### 4.3 트리거 정책
- eTAS 기본 정책: `DAILY_CLOSE`
- 필요 시 일부 차량/고객 요구에 따라 `TRIP_ENDED` 보조 흐름을 허용할 수 있으나,
  최종 제출 단위와 멱등 기준은 일자 단위 정책과 충돌하지 않도록 별도 관리한다.

---

## 5) eTAS 제출용 데이터 구성

### 5.1 원천 데이터 소스
- 실시간 telemetry / trip 데이터
- 차량 마스터(Admin Web)
- 사업자 마스터(Admin Web)
- 운전자 마스터 및 차량-운전자 배정 이력(Admin Web)

### 5.2 eTAS 제출 시 필요한 보강 메타
아래 항목은 단말 payload만으로 부족할 수 있으므로 서버가 기준정보로 보강한다.

- 자동차등록번호
- 차대번호(VIN)
- 자동차 유형 코드
- 운송사업자 등록번호
- 운전자코드
- DTG 장치 모델명
- 일일주행거리
- 누적주행거리
- 파일명 생성용 일자/차량 식별 정보

### 5.3 데이터 책임 분리
- 단말: 위치/속도/RPM/브레이크/방위각/누적거리 등 원천 주행 데이터
- Admin Web: 차량/사업자/운전자 기준정보
- 서버: 일자 집계, 포맷 변환, 법규 검증, eTAS 전송

---

## 6) 라우팅 규칙

### 6.1 공통 라우팅
1. `vehicle_device_binding` 또는 차량 매핑 정보에서 `outbound_profile_id`, `outbound_trigger_policy` 조회
2. 트리거 정책 불일치 시 skip
3. `outbound_interface_catalog`에서 대상 인터페이스 목록 조회
4. `priority` 오름차순으로 전송 시도
5. 인터페이스별 mapper / connector / adapter 실행

### 6.2 eTAS 기본 라우팅 원칙
- eTAS 인터페이스 코드는 별도 catalog로 관리한다.
- eTAS는 현재 우선 외부 연계 대상이므로 profile에서 명시적으로 활성화되어야 한다.
- 차량별로 eTAS 전송 대상 여부를 제어할 수 있어야 한다.

---

## 7) eTAS 변환 및 검증 규칙

### 7.1 변환 원칙
- 내부 JSON/집계 모델을 eTAS 제출 포맷으로 변환한다.
- 시간값은 eTAS 요구 형식으로 변환한다.
- 좌표, 등록번호, 코드값은 eTAS 규격의 자릿수/표현 규칙에 맞춘다.
- 파일명은 `[별표 3]` 규칙을 따른다.

### 7.2 주요 검증 항목
- 자동차등록번호 표기 규칙 검증
- 차대번호 길이/문자셋 검증
- 자동차 유형 코드셋 검증
- 운송사업자 등록번호 형식 검증
- 운전자코드 존재 여부 및 형식 검증
- 일일주행거리/누적주행거리 정합성 검증
- 운행기록 필드 배열순서 및 누락 여부 검증

### 7.3 검증 실패 처리
- 검증 실패 시 eTAS 전송을 수행하지 않는다.
- `outbound_transmission`에는 실패 상태와 상세 오류코드를 저장한다.
- 운영자 재검토가 필요한 경우 알림 또는 보정 큐로 분기한다.

---

## 8) 멱등키 규칙

### 8.1 공통 포맷
- `idempotencyKey = interfaceCode / triggerPolicy / tenantId / vehicleId / resourceKey`

### 8.2 eTAS resourceKey
- `DAILY_CLOSE`: `YYYY-MM-DD`
- `TRIP_ENDED`: 보조 흐름에서만 `tripId` 사용 가능

### 8.3 eTAS 기본 멱등 기준
- eTAS는 기본적으로 **차량 / 일자 / 인터페이스** 조합을 동일 전송 단위로 본다.
- 동일 키가 `SENT`이면 재전송 차단
- 동일 키가 `FAILED(retryable)`이면 재시도 허용

### 8.4 중복 처리
- `outbound_transmission.idempotency_key` UNIQUE
- 중복 전송 방지와 운영 재처리 구분이 가능해야 한다.

---

## 9) 재시도 / DLQ 정책

### 9.1 재시도 대상
- timeout
- network error
- 5xx 응답
- 일시적 외부 인터페이스 장애

### 9.2 재시도 제외
- validation error(4xx)
- 기준정보 누락/정합성 오류
- 인증 오류(명시적 차단)

### 9.3 백오프
- 1m -> 5m -> 15m -> 1h -> 6h (최대 5회)

### 9.4 DLQ
- 최대 시도 초과 시 DLQ 적재
- 운영자 수동 재처리 API 또는 관리자 화면에서 재실행

### 9.5 eTAS 특화 재처리 기준
- 외부 시스템 일시 장애: 자동 재시도
- 차량/운전자/사업자 메타 누락: 자동 재시도 대상 아님
- 기준정보 보정 후 운영자 재처리 필요

---

## 10) 처리 결과 저장

`outbound_transmission`에 아래 저장:
- `status` (`PENDING`/`SENT`/`FAILED`)
- `attempt_count`
- `last_error_code`
- `request_payload_json` 또는 생성 전문 메타
- `response_payload_json`
- `trigger_policy`
- `outbound_profile_id`
- `interface_code`
- `vehicle_id`
- `business_date`
- `idempotency_key`

추가로 eTAS 전송 추적을 위해 아래 저장을 권장한다.
- `generated_file_name`
- `validation_result_json`
- `master_snapshot_ref` 또는 기준정보 버전 참조

---

## 11) 상태 전이

- `PENDING -> SENT`
- `PENDING -> FAILED(retryable)`
- `FAILED(retryable) -> PENDING(retry)`
- `FAILED(non-retryable) -> FAILED(final)`
- `FAILED(validation) -> FAILED(final)`

---

## 12) 예외 처리

- profile 없음/비활성: `OUTBOUND_PROFILE_INVALID`
- 인터페이스 없음/비활성: `OUTBOUND_INTERFACE_NOT_AVAILABLE`
- mapper 실패: `OUTBOUND_MAPPING_ERROR`
- 전송 timeout: `OUTBOUND_TIMEOUT`
- 차량 메타 누락: `OUTBOUND_ETAS_VEHICLE_METADATA_MISSING`
- 사업자 메타 누락: `OUTBOUND_ETAS_BIZ_METADATA_MISSING`
- 운전자 메타 누락: `OUTBOUND_ETAS_DRIVER_METADATA_MISSING`
- 파일명 규칙 위반: `OUTBOUND_ETAS_FILENAME_INVALID`
- 법규 정합성 검증 실패: `OUTBOUND_ETAS_VALIDATION_FAILED`

---

## 13) 관측 지표

- outbound sent/failed count by interface
- retry count by interface
- DLQ count
- trigger별 처리량(`TRIP_ENDED`/`DAILY_CLOSE`)
- end-to-end latency (trigger -> sent)
- eTAS validation fail count
- metadata missing count by type(차량/사업자/운전자)
- business_date 기준 전송 완료율

---

## 14) 테스트 최소셋

1. `DAILY_CLOSE` 기준 eTAS 정상 전송
2. `TRIP_ENDED` 보조 흐름 사전 생성/검증
3. 차량 메타 누락 시 validation fail
4. 운전자 메타 누락 시 validation fail
5. 파일명 생성 규칙 검증
6. idempotency 중복 차단
7. 외부 장애 시 retry -> DLQ 흐름 검증
8. 기준정보 수정 후 수동 재처리 성공 검증

---

## 15) 운영 메모

- eTAS 제출 데이터는 단말 payload 원문만으로 생성되지 않는다.
- Admin Web에서 차량/사업자/운전자 기준정보를 사전 관리해야 한다.
- eTAS 전송 실패 중 기준정보 누락 유형은 운영자 보정 프로세스와 연결되어야 한다.
- 향후 다른 외부 연계가 추가되더라도, eTAS 관련 법규 검증 로직은 별도 모듈로 유지하는 것이 바람직하다.

---

## 16) 변경 이력

- v2.0 (2026-03-25)
  - 보조 트리거 `TRIP_ENDED` 이벤트: Kinesis → SQS 변경
  - TrackCar 플랫폼 아키텍처 상세 설계 v3.0.0 기준 적용
- v1.1 (2026-03-17): 제목을 `TrackCar 외부 연계 전송 처리 명세서`로 변경하고, eTAS 우선 범위/기준정보 보강/법규 검증/운영 재처리 규칙을 구체화
- v1.0 (2026-02-13): 최초 작성

#업무/프로젝트/trackcar/산출물
