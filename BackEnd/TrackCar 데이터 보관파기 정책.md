# TrackCar 데이터 보관/파기 정책
- 작성일: 2026-02-13
- 목적: 위치/운행 데이터의 보관기간을 명확히 정의하고 자동 파기 기준을 운영 표준으로 고정
- 적용 범위: TrackCar 수집/저장/조회/연계 전 구간(AWS)

---

## 1) 정책 원칙

1. 위치/운행 관련 원천 데이터는 **최대 3년** 보관한다.
2. 3년 경과 데이터는 자동 파기한다.
3. DynamoDB 실시간 테이블은 장기보관소가 아닌 운영 캐시로 운영한다.
4. 과금/정산/감사 데이터는 별도 규정이 우선하며 본 정책의 예외로 분리 관리할 수 있다.

---

## 2) 데이터 분류별 보관/파기 기준

| 분류 | 저장소 | 보관기간 | 파기 방식 |
|---|---|---:|---|
| DTG Raw 위치 원본 | S3 Raw | 3년 | S3 Lifecycle Expiration |
| Trip 경로 파일 | S3 Trip Track | 3년 | S3 Lifecycle Expiration |
| 운행 메타(trip/alert/운행이력) | Aurora PostgreSQL | 3년 | 배치 삭제(Purge Job) |
| 실시간 현재 위치(`vehicle_current_state`) | DynamoDB | 최신 상태 유지(이력 미보관) | Upsert + 장기 미수신 TTL 정리 |
| 리플레이/멱등성 키 | DynamoDB | 단기(1~7일) | TTL 자동 삭제 |
| 리포트 산출 파일 | S3 Report | 30일 | S3 Lifecycle Expiration |

> 기준일: `event_time` 또는 `trip_date` 기준으로 3년 경과 여부 판단

---

## 3) 구현 정책

## 3.1 S3 Lifecycle
- Raw Bucket: `Expiration = 1095일`
- Trip Track Bucket: `Expiration = 1095일`
- Report Bucket: `Expiration = 30일`

## 3.2 Aurora Purge Job
- 실행 주기: 일 1회(권장 02:00 KST)
- 삭제 조건: `trip_date < (current_date - interval '3 years')`
- 삭제 대상 예시:
  - `trip_meta`
  - `alert_event`
  - 위치/운행 이력성 보조 테이블
- 삭제 로그: `ops_data_purge_log`에 건수/실패사유 기록

## 3.3 DynamoDB TTL
- `vehicle_current_state`: 최신값 upsert(기본 TTL 없음), 선택적으로 마지막 수신 후 90일 미수신 차량 정리 TTL 적용 가능
- `idempotency_store`/`replay_nonce`: TTL 필수(권장 1~7일)

---

## 4) 운영/감사

- 모니터링 지표
  - S3 Lifecycle 만료 건수
  - Purge Job 성공/실패 횟수
  - Purge 삭제 레코드 수
  - TTL 만료 처리 지연 건수
- 월 1회 샘플 검증
  - 3년 초과 데이터 잔존 여부 점검
  - 파기 로그와 실제 데이터 일치 여부 점검

---

## 5) 예외/보류 정책

다음 조건에서는 파기를 일시 보류할 수 있다.
- 법적 분쟁/감사 대응을 위한 보존 요청(legal hold)
- 운영 장애로 인한 일시 보류(최대 보류 기간/승인자 기록 필수)

보류 해제 후 누락된 파기 작업은 즉시 재실행한다.

---

## 6) 연계 정책

- ETAS/외부 전송용 운영 이력(`outbound_transmission`, `etas_transmission`)은 별도 규정이 없는 경우 3년 이내 관리 권장
- 단, 회계/세무/전자금융 관련 데이터는 법적 최소 보관기간이 우선한다.

---

## 7) 변경 이력

- v1.0 (2026-02-13):
  - 위치 원본 S3 및 운행이력 3년 보관/파기 기준 최초 확정
  - DynamoDB 실시간/멱등성 데이터 단기 보관 원칙 명시


---

## v2.4.0 정합화 업데이트 (2026-03-02)

본 문서는 **TrackCar 아키텍처 심화 설계 v2.4.0** 기준으로 해석/구현되어야 한다.

### 공통 반영 기준
- 실시간 수집 경로: **AWS IoT Core(MQTT/mTLS) → IoT Rules → Kinesis(raw-ingest) → ingest-auth-validate → Kinesis(clean-stream)**
- 비즈니스 처리: `telemetry-process`가 **DynamoDB 최신 상태 Upsert + Aurora 이벤트/트립 적재** 수행
- Cold 데이터: **Firehose → S3 Parquet** 저장(경로 파티셔닝, Lifecycle 3년)
- eTAS: **실시간 분리형 심야 배치**(EventBridge + Step Functions + Formatter + 재시도/로그)
- API 계층: Client/Partner API는 **공용 Business Service**를 호출(도메인 로직 단일화)

### 구현 시 강제 체크
- 멱등키: `deviceId + batchId` 기준 Dedup Guard(DynamoDB TTL)
- RDB 연결: **RDS Proxy 경유**
- 포맷 준수: eTAS `[별표 2]` 배열/자릿수, `[별표 3]` 파일명 규격
- 장애 대응: 지수 백오프 재시도 + 전송로그(`etas_transmission_log`) 기록

> 상세 기준 문서: `TrackCar 아키텍처 심화 설계` (v2.4.0, 2026-03-02)

#업무/프로젝트/trackcar/산출물