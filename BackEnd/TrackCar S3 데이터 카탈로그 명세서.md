# TrackCar S3 데이터 카탈로그 명세서

- 작성일: 2026-03-25
- 버전: v2.0
- 상태: 초안
- 기반: TrackCar 플랫폼 아키텍처 상세 설계 v3.0.0

---

## 1. 문서 개요

### 1.1 목적

본 문서는 TrackCar 백엔드의 Cold Data 저장소인 Amazon S3의 데이터 카탈로그를 정의한다. 버킷 구조, 경로 패턴, 파티션 전략, 오브젝트 스키마, Lifecycle 정책을 명시하여 데이터 저장 및 분석의 기준을 제공한다.

> **아키텍처 변경 (v3.0.0):** Kinesis Data Firehose → Lambda 직접 S3 Parquet 적재로 변경. Cold 데이터는 `telemetry-process` Lambda가 실시간으로 S3에 직접 적재한다.

### 1.2 버킷 구성

| 버킷 | 용도 | 리전 | 암호화 |
|-------|------|------|--------|
| `trackcar-raw-telemetry` | DTG 원본 텔레메트리 Parquet | ap-northeast-2 | SSE-S3 |
| `trackcar-trips` | 운행 경로 (Trip Track) | ap-northeast-2 | SSE-S3 |
| `trackcar-reports` | 리포트/산출물 | ap-northeast-2 | SSE-S3 |
| `trackcar-etl-staging` | ETL 임시 파일 | ap-northeast-2 | SSE-S3 |

### 1.3 설계 원칙

- **파티션 기반 경로**: `date`, `hour`, `deviceId` 기준 파티셔닝
- **Parquet 포맷**: 분석 효율성 및 압축률 최적화
- **Lifecycle Expiration**: 3년(1095일) 후 자동 파기
- **버저닝**: 비활성화 (불필요한 비용 증가 방지)
- **Object Lock**: 비활성화 (삭제/수정 자주 발생)

---

## 2. Raw Telemetry 버킷 (trackcar-raw-telemetry)

### 2.1 목적

Lambda(`telemetry-process`)가 실시간으로 직접 Parquet 변환 후 적재. eTAS 전송 원본 및 분석용 데이터 소스로 활용.

### 2.2 경로 구조

```
s3://trackcar-raw-telemetry/
├── raw/
│   └── date=YYYY-MM-DD/
│       └── hour=HH/
│           └── deviceId={deviceId}/
│               └── part-{uuid}.parquet
├── processed/
│   └── date=YYYY-MM-DD/
│       └── hour=HH/
│           └── deviceId={deviceId}/
│               └── part-{uuid}.parquet
└── etas-source/
    └── date=YYYY-MM-DD/
        └── deviceId={deviceId}/
            └── part-{uuid}.parquet
```

### 2.3 Parquet 스키마

```protobuf
message DtgTelemetry {
  // 메시지 식별
  required string message_id = 1;        // UUID
  required string device_id = 2;          // 장치 ID
  required string batch_id = 3;            // 배치 ID (멱등키)
  required int64  sent_at = 4;             //发送 시각 (Unix timestamp ms)
  required int64  received_at = 5;         // 수신 시각 (Unix timestamp ms)
  
  // 위치 정보
  required double latitude = 10;           // 위도 (-90 ~ 90)
  required double longitude = 11;          // 경도 (-180 ~ 180)
  required int64  recorded_at = 12;        // 기록 시각 (Unix timestamp ms)
  
  // 차량 상태
  required boolean ignition = 20;          // 시동 상태
  optional int32   speed = 21;             // 속도 (km/h)
  optional int32   heading = 22;           // 방위각 (0-360°)
  optional int32   rpm = 23;               // 엔진 RPM
  optional int32   fuel_level = 24;        // 연료 잔량 (%)
  optional int32   odometer = 25;          // 누적 주행거리 (km)
  
  // 브레이크/감속
  optional bool    brake_status = 30;      // 브레이크 상태
  optional int32   brake_pressure = 31;    // 브레이크 압력
  
  // 메타데이터
  optional string  interface_code = 40;    // 인터페이스 코드
  optional string  interface_version = 41; // 인터페이스 버전
  optional string  vehicle_id = 42;         // 차량 ID (매핑 후)
  optional string  tenant_id = 43;          // 테넌트 ID
  optional string  raw_payload = 44;        // 원본 페이로드 (JSON)
}
```

### 2.4 Parquet 파일 속성

| 속성 | 값 |
|------|-----|
| Format | Parquet |
| Compression | GZIP |
| Row Group Size | 128MB |
| Page Size | 1MB |
| Version | 2.0 |
| Writer | Lambda (telemetry-process) 직접 적재 |

---

## 3. Trip Track 버킷 (trackcar-trips)

### 3.1 목적

운행(Trip) 경로 데이터 저장. 출발/도착 좌표, 경유 포인트, 이벤트 포함.

### 3.2 경로 구조

```
s3://trackcar-trips/
├── trips/
│   └── date=YYYY-MM-DD/
│       └── tripId={tripId}/
│           ├── trip.parquet              -- Trip 메타 + 포인트
│           └── events.parquet            -- Trip 이벤트
└── aggregated/
    └── date=YYYY-MM-DD/
        └── hour=HH/
            └── part-{uuid}.parquet       -- 집계 데이터
```

### 3.3 Trip Parquet 스키마

```protobuf
message TripRecord {
  // Trip 식별
  required string trip_id = 1;
  required string vehicle_id = 2;
  optional string driver_id = 3;
  required int64  started_at = 4;
  optional int64  ended_at = 5;
  
  // 위치 정보
  optional double start_lat = 10;
  optional double start_lng = 11;
  optional double end_lat = 12;
  optional double end_lng = 13;
  repeated LocationPoint route_points = 20;
  
  // 통계
  optional double total_distance_km = 30;
  optional int32  total_duration_min = 31;
  optional int32  max_speed = 32;
  optional int32  avg_speed = 33;
  
  // 상태
  required string status = 40;            // RUNNING, COMPLETED, CANCELLED
  required string trip_date = 41;         // YYYY-MM-DD
}

message LocationPoint {
  required double latitude = 1;
  required double longitude = 2;
  required int64  recorded_at = 3;
  optional int32  speed = 4;
  optional int32  heading = 5;
  optional bool   ignition = 6;
}
```

### 3.4 Events Parquet 스키마

```protobuf
message TripEvent {
  required string event_id = 1;
  required string trip_id = 2;
  required string vehicle_id = 3;
  required string event_type = 4;         // IGNITION_ON, IGNITION_OFF, STOP, ALERT
  required int64  event_time = 5;
  optional double latitude = 6;
  optional double longitude = 7;
  optional string description = 8;
  optional int32  duration_min = 9;       // STOP 이벤트 시 정차 시간
  optional string alert_severity = 10;    // ALERT 이벤트 시
}
```

---

## 4. Reports 버킷 (trackcar-reports)

### 4.1 목적

일일/주간/월간 리포트 파일 저장. Operator 관리용.

### 4.2 경로 구조

```
s3://trackcar-reports/
├── daily/
│   └── date=YYYY-MM-DD/
│       └── organizationId={orgId}/
│           └── report-{type}-{date}.pdf
├── weekly/
│   └── week=YYYY-WW/
│       └── organizationId={orgId}/
│           └── report-{type}-{week}.pdf
├── monthly/
│   └── month=YYYY-MM/
│       └── organizationId={orgId}/
│           └── report-{type}-{month}.pdf
└── exports/
    └── date=YYYY-MM-DD/
        └── userId={userId}/
            └── export-{uuid}.csv
```

### 4.3 Lifecycle 정책

```
Expiration: 30일
```

---

## 5. ETL Staging 버킷 (trackcar-etl-staging)

### 5.1 목적

eTAS 전송 포맷 변환을 위한 임시 파일 저장. S3 → Lambda/Glue → eTAS 처리 중 임시 저장.

### 5.2 경로 구조

```
s3://trackcar-etl-staging/
├── etas/
│   └── date=YYYY-MM-DD/
│       └── vehicleId={vehicleId}/
│           ├── raw.parquet               -- 원본 Parquet
│           └── formatted.txt             -- eTAS 포맷 TXT (별표 2)
└── temp/
    └── jobId={jobId}/
        └── *.parquet
```

### 5.3 Lifecycle 정책

```
Expiration: 7일
```

---

## 6. eTAS 전송 파일 명세

### 6.1 [별표 3] 파일명 규격

```
{YYYYMMDD}-{AAAAAAAAAAAAAA}-B-{CC}-{DDDDDDD}.TXT
```

| 위치 | 길이 | 설명 | 예시 |
|------|------|------|------|
| YYYYMMDD | 8 | 데이터 생성일자 | 20260325 |
| AAAAAAAAAAAAAA | 14 | 운송사업자등록번호 | 12345678901234 |
| B | 1 | 파일구분 (B=사업용) | B |
| CC | 2 | 파일일련번호 (01~99) | 01 |
| DDDDDDD | 7 | 자동차등록번호 | 12가3456 |

**예시:**
```
20260325-12345678901234-B-01-12가3456.TXT
```

### 6.2 [별표 2] 파일 내용 규격

```
[항번] [항목] [구분] [자리수] [값] [빈칸|#]
```

| 항목 | 내용 | 자리수 | 설명 |
|------|------|--------|------|
| 1 | 자동차등록번호 | 12 | 차량번호 |
| 2 | 차대번호 | 17 | VIN |
| 3 | 데이터생성일시 | 14 | YYYYMMDDHHmmss |
| 4 | 운전자코드 | 20 | 운전자코드 |
| 5 | 최초동일시각 | 14 | YYYYMMDDHHmmss |
| 6 | 최종동일시각 | 14 | YYYYMMDDHHmmss |
| 7 | 일일주행거리 | 7 | km (소수점 1자리) |
| 8 | 누적주행거리 | 7 | km (소수점 1자리) |
| 9 | 운행기록수 | 6 | 레코드 수 |
| 10 | 운행시작시각 | 14 | YYYYMMDDHHmmss |
| 11 | 운행종료시각 | 14 | YYYYMMDDHHmmss |
| 12~ | 운행기록 1~N | 가변 | 1분 단위 데이터 |

**운행기록 형식:**
```
위도(10),경도(11),속도(4),방위각(4),브레이크(1),시간(14)
```

---

## 7. Lifecycle 정책

### 7.1 버킷별 Lifecycle

| 버킷 | 용도 | Expiration | 스토리지 클래스 전환 |
|-------|------|-----------|---------------------|
| trackcar-raw-telemetry | 원본 텔레메트리 | 1095일 (3년) | 365일 후 → S3 Glacier |
| trackcar-trips | 운행 경로 | 1095일 (3년) | 365일 후 → S3 Glacier |
| trackcar-reports | 리포트 | 30일 | - |
| trackcar-etl-staging | ETL 임시 | 7일 | - |

### 7.2 S3 Lifecycle Rule 설정

```json
{
  "Rules": [
    {
      "ID": "raw-telemetry-expiration",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "raw/"
      },
      "Expiration": {
        "Days": 1095
      },
      "Transitions": [
        {
          "Days": 365,
          "StorageClass": "GLACIER"
        }
      ]
    },
    {
      "ID": "trips-expiration",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "trips/"
      },
      "Expiration": {
        "Days": 1095
      },
      "Transitions": [
        {
          "Days": 365,
          "StorageClass": "GLACIER"
        }
      ]
    },
    {
      "ID": "reports-cleanup",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "reports/"
      },
      "Expiration": {
        "Days": 30
      }
    },
    {
      "ID": "etl-staging-cleanup",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "etl-staging/"
      },
      "Expiration": {
        "Days": 7
      }
    }
  ]
}
```

---

## 8. 데이터 파티셔닝 가이드

### 8.1 Athena 파티션 정의

```sql
-- raw-telemetry 테이블
CREATE EXTERNAL TABLE raw_telemetry (
  message_id STRING,
  device_id STRING,
  batch_id STRING,
  sent_at BIGINT,
  received_at BIGINT,
  latitude DOUBLE,
  longitude DOUBLE,
  recorded_at BIGINT,
  ignition BOOLEAN,
  speed INT,
  heading INT,
  rpm INT,
  fuel_level INT,
  odometer INT,
  interface_code STRING,
  interface_version STRING,
  vehicle_id STRING,
  tenant_id STRING
)
PARTITIONED BY (date STRING, hour STRING, device_id_path STRING)
STORED AS PARQUET
LOCATION 's3://trackcar-raw-telemetry/raw/';

-- 파티션 업데이트
MSCK REPAIR TABLE raw_telemetry;
```

### 8.2 Glue Data Catalog

```json
{
  "TableInput": {
    "Name": "raw_telemetry",
    "StorageDescriptor": {
      "Location": "s3://trackcar-raw-telemetry/raw/",
      "InputFormat": "org.apache.hadoop.mapred.TextInputFormat",
      "OutputFormat": "org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat",
      "SerdeInfo": {
        "SerializationLibrary": "org.apache.hadoop.hive.ql.io.parquet.serde.ParquetHiveSerDe"
      }
    },
    "PartitionKeys": [
      {"Name": "date", "Type": "string"},
      {"Name": "hour", "Type": "string"},
      {"Name": "device_id_path", "Type": "string"}
    ],
    "TableType": "EXTERNAL_TABLE"
  }
}
```

---

## 9. 보안 설정

### 9.1 버킷 정책

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RestrictAccess",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::trackcar-raw-telemetry/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    }
  ]
}
```

### 9.2 암호화

| 설정 | 값 |
|------|-----|
| Default Encryption | SSE-S3 (AES-256) |
| CloudTrail | 활성화 |
| Access Logging | 활성화 |

### 9.3 접근 제어

| 역할 | 권한 |
|------|------|
| Lambda (telemetry-process) | Write (raw-telemetry) |
| Lambda (eTAS Formatter) | Read (raw-telemetry), Write (etl-staging) |
| Glue/Athena | Read |
| Admin Users | Read (CloudWatch Logs) |

---

## 10. 모니터링

### 10.1 CloudWatch Metrics

| 지표 | 임계값 | 알람 |
|------|--------|------|
| BucketSizeBytes | - | 주간 보고 |
| NumberOfObjects | - | 주간 보고 |
| AllRequests | - | - |
| 4xxErrors | > 0 | WARNING |
| 5xxErrors | > 0 | CRITICAL |

### 10.2 S3 Inventory

| 설정 | 값 |
|------|-----|
| Frequency | Daily |
| Format | Parquet |
| Destination | s3://trackcar-reports/inventory/ |
| Fields | Key, Size, LastModifiedDate, ETag, StorageClass |

---

## 11. 변경 이력 (Changelog)

- **v2.0 (2026-03-25):**
  - Kinesis Data Firehose → Lambda 직접 S3 Parquet 적재 변경
  - Writer: `Kinesis Data Firehose` → `Lambda (telemetry-process) 직접 적재`
  - Compression: `Snappy` → `GZIP`
  - Kinesis Firehose 관련 역할 권한 제거
  - TrackCar 플랫폼 아키텍처 상세 설계 v3.0.0 기준 적용

- **v1.1 (2026-03-25):**
  - AWS 리전 `ap-northeast-2` (서울) 명시

- **v1.0 (2026-03-25):**
  - 최초 데이터 카탈로그 명세서 작성
  - TrackCar 플랫폼 아키텍처 상세 설계 v2.4.0 기준 적용
  - 버킷 4개 정의 (raw-telemetry, trips, reports, etl-staging)
  - Parquet 스키마 정의 (원본 텔레메트리, Trip, Events)
  - Lifecycle 정책 정의 (3년/30일/7일)
  - eTAS 파일명/내용 규격 포함 ([별표 2], [별표 3])

---

## 12. 참고 문서

| 문서명 | 버전 | 요약 |
|--------|------|------|
| TrackCar 플랫폼 아키텍처 상세 설계 | v3.0.0 | 전체 시스템 아키텍처 |
| TrackCar 외부 연계 전송 처리 명세서 | v1.1 | eTAS 전송 처리 |
| TrackCar 데이터 보관파기 정책 | v1.0 | Lifecycle 기준 |
| [별표 2] 운행기록의 배열순서 | - | TXT 파일 포맷 |
| [별표 3] 운행기록의 파일명 | - | TXT 파일명 규격 |
