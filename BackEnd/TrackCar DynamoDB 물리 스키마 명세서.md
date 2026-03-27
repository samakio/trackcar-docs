# TrackCar DynamoDB 물리 스키마 명세서

- 작성일: 2026-03-25
- 버전: v1.0
- 상태: 초안
- 기반: TrackCar 플랫폼 아키텍처 상세 설계 v2.4.0

---

## 1. 문서 개요

### 1.1 목적

본 문서는 TrackCar 백엔드의 Hot Data 저장소인 Amazon DynamoDB의 물리 스키마를 정의한다. 테이블 구조, 키 설계, 인덱스, TTL 정책을 명시하여 실시간 데이터 처리 및 조회 성능을 최적화한다.

### 1.2 설계 원칙

- **단일 테이블 설계**: 액세스 패턴 기반 테이블 분리 최소화
- **PK/SK 설계**: 향후 GSI 확장성을 고려한 키 구조
- **TTL 활용**: Hot Data는 최소 TTL 설정으로 자동 정리
- **On-Demand Capacity**: 초기 트래픽 예측困难的 경우 On-Demand 모드 권장

### 1.3 테이블 구성

| 테이블 | 용도 | 리전 | 프로비저닝 |
|--------|------|------|-----------|
| VehicleLatestLocation | 차량 최신 위치 (실시간 대시보드) | ap-northeast-2 (서울) | On-Demand |
| IngestDedupGuard | 수집 멱등성 보장 | ap-northeast-2 (서울) | On-Demand |
| SessionStore | 사용자 세션 관리 | ap-northeast-2 (서울) | Provisioned |

---

## 2. VehicleLatestLocation 테이블

### 2.1 목적

실시간 대시보드 렌더링 및 중복 방지를 위한 차량 최신 위치 정보 저장.

### 2.2 스키마

```
Table: VehicleLatestLocation
Billing: On-Demand
```

| 속성 | 타입 | 키 | 설명 |
|------|------|-----|------|
| PK | String | PK | `VEHICLE#{vehicleId}` |
| SK | String | SK | `STATE` |
| vehicleId | String | - | 차량 UUID |
| vehicleNo | String | - | 차량번호 |
| lat | Number | - | 위도 |
| lng | Number | - | 경도 |
| speed | Number | - | 속도 (km/h) |
| heading | Number | - | 방위각 (0-360°) |
| ignition | Boolean | - | 시동 상태 |
| driverId | String | - | 연결된 기사 ID |
| driverName | String | - | 연결된 기사명 |
| groupId | String | - | 소속 그룹 ID |
| groupName | String | - | 소속 그룹명 |
| organizationId | String | - | 소속 조직 ID |
| tripStatus | String | - | 운행 상태 (RUNNING, IDLE, OFFLINE) |
| deviceStatus | String | - | 장치 상태 (ONLINE, OFFLINE) |
| lastReceivedAt | String | - | 마지막 수신 시각 (ISO8601) |
| updatedAt | String | - | 최종 업데이트 시각 (ISO8601) |
| TTL | Number | TTL | TTL (unixtime, 2일 후 만료) |

### 2.3 GSI (Global Secondary Index)

**GSI-1: OrganizationIndex**
```
IndexName: GSI-OrganizationIndex
Partition Key: organizationId
Sort Key: SK
Projection: ALL
```

| 속성 | 타입 | 설명 |
|------|------|------|
| organizationId | String | 조직 ID |
| SK | String | `STATE` |
| vehicleId | String | 차량 UUID |
| vehicleNo | String | 차량번호 |
| lat | Number | 위도 |
| lng | Number | 경도 |
| tripStatus | String | 운행 상태 |
| ignition | Boolean | 시동 상태 |
| updatedAt | String | 최종 업데이트 |

**GSI-2: GroupIndex**
```
IndexName: GSI-GroupIndex
Partition Key: groupId
Sort Key: updatedAt
Projection: ALL
```

**GSI-3: TripStatusIndex**
```
IndexName: GSI-TripStatusIndex
Partition Key: tripStatus
Sort Key: updatedAt
Projection: ALL
```

**GSI-4: DeviceStatusIndex**
```
IndexName: GSI-DeviceStatusIndex
Partition Key: deviceStatus
Sort Key: updatedAt
Projection: ALL
```

### 2.4 액세스 패턴

| 패턴 | PK | SK | 인덱스 | 설명 |
|------|----|----|--------|------|
| 단일 차량 최신 위치 | VEHICLE#{id} | STATE | - | 차량별 실시간 위치 조회 |
| 조직별 차량 목록 | - | - | GSI-OrganizationIndex | 조직 전체 차량 현황 |
| 그룹별 차량 목록 | - | - | GSI-GroupIndex | 그룹 내 차량 현황 |
| 운행중 차량 목록 | - | - | GSI-TripStatusIndex | RUNNING 상태 차량 |
| 오프라인 차량 목록 | - | - | GSI-DeviceStatusIndex | OFFLINE 상태 장치 |

### 2.5 TTL 정책

```
TTL: lastReceivedAt + 172800 (2일)
```

- 마지막 수신 후 2일 경과 시 자동 삭제
- DynamoDB Streams로 삭제 이벤트捕獲하여 Aurora 백업 가능

---

## 3. IngestDedupGuard 테이블

### 3.1 목적

수집 파이프라인에서 `deviceId + batchId` 기준 멱등성 보장 및 중복 방지.

### 3.2 스키마

```
Table: IngestDedupGuard
Billing: On-Demand
```

| 속성 | 타입 | 키 | 설명 |
|------|------|-----|------|
| PK | String | PK | `KEY#{deviceId}#{batchId}` |
| SK | String | SK | `DEDUP` |
| deviceId | String | - | 장치 ID |
| batchId | String | - | 배치 ID (멱등키) |
| processedAt | String | - | 처리 시각 (ISO8601) |
| status | String | - | PROCESSED, FAILED |
| TTL | Number | TTL | TTL (1~7일) |

### 3.3 TTL 정책

```
TTL: processedAt + 604800 (7일, 최대)
```

- 기본값: 1일 (86400초)
- 운영 환경에 따라 조정 가능

### 3.4 액세스 패턴

| 패턴 | PK | SK | 설명 |
|------|----|----|------|
| 중복 체크 | KEY#{deviceId}#{batchId} | DEDUP | GetItem으로 존재 여부 확인 |

### 3.5 ConditionExpression

```javascript
// 저장 시 조건
ConditionExpression: attribute_not_exists(PK)

// 저장 후 결과
{
  "Attributes": {
    "PK": "KEY#dev_001#batch_20260325_001",
    "SK": "DEDUP",
    "deviceId": "dev_001",
    "batchId": "batch_20260325_001",
    "processedAt": "2026-03-25T10:00:00Z",
    "status": "PROCESSED",
    "TTL": 1742976000
  }
}
```

---

## 4. SessionStore 테이블

### 4.1 목적

AWS Cognito 세션 외에 애플리케이션 레벨 세션/토큰 관리.

### 4.2 스키마

```
Table: SessionStore
Billing: Provisioned (RCU: 10, WCU: 10)
```

| 속성 | 타입 | 키 | 설명 |
|------|------|-----|------|
| PK | String | PK | `SESSION#{sessionId}` |
| SK | String | SK | `META` |
| userId | String | - | 사용자 ID |
| userType | String | - | OWNER, STAFF, DRIVER |
| organizationId | String | - | 조직 ID |
| refreshToken | String | - | 리프레시 토큰 |
| deviceId | String | - | 디바이스 ID |
| createdAt | String | - | 생성 시각 |
| expiresAt | String | - | 만료 시각 |
| TTL | Number | TTL | TTL (만료 시간) |

### 4.2 GSI (Global Secondary Index)

**GSI-1: UserSessionIndex**
```
IndexName: GSI-UserSessionIndex
Partition Key: userId
Sort Key: createdAt
Projection: ALL
```

### 4.3 TTL 정책

```
TTL: expiresAt (unixtime)
```

- 세션 만료 시 자동 삭제

---

## 5. DynamoDB Streams 설정

### 5.1 VehicleLatestLocation Streams

| 설정 | 값 |
|------|-----|
| Stream View Type | NEW_AND_OLD_IMAGES |
| 목적 | 삭제 이벤트捕獲 → Aurora 동기화 백업 |

### 5.2 구현 예시 (EventBridge + Lambda)

```javascript
// Lambda: DynamoDB Stream → Aurora Sync
exports.handler = async (event) => {
    for (const record of event.Records) {
        if (record.eventName === 'REMOVE') {
            const vehicleId = record.dynamodb.Keys.PK.S.replace('VEHICLE#', '');
            const oldData = record.dynamodb.OldImage;
            
            // Aurora에 백업 또는 삭제 로그 기록
            await syncToAurora(vehicleId, oldData);
        }
    }
};
```

---

## 6. 온보딩/인증 관련 (IoT Core 연계)

### 6.1 JITR 인증서 관리용 테이블

DynamoDB는 IoT Core JITR 과정에서 필요한 인증서 메타데이터 임시 저장에 활용.

```
Table: DeviceCertificateMetadata
Billing: On-Demand
```

| 속성 | 타입 | 키 | 설명 |
|------|------|-----|------|
| PK | String | PK | `CERT#{certificateId}` |
| SK | String | SK | `META` |
| deviceId | String | - | 장치 ID |
| certificateId | String | - | 인증서 ID (IoT Core) |
| certificateArn | String | - | 인증서 ARN |
| status | String | - | PENDING_ACTIVATION, ACTIVE, INACTIVE |
| thingName | String | - | IoT Thing 이름 |
| createdAt | String | - | 생성 시각 |
| activatedAt | String | - | 활성화 시각 |
| TTL | Number | TTL | TTL (30일) |

### 6.2 TTL 정책

```
TTL: activatedAt + 2592000 (30일, 활성화 후)
```

---

## 7. 공통 설계 패턴

### 7.1 NULL 값 처리

- 빈 문자열 (`""`) 대신 `NULL` 사용
- 선택적 속성은 항상 `NULL`로 표현

### 7.2 숫자 타입

- 모든 숫자는 `Number` 타입 (DynamoDB 내장)
- 정밀도 요구 시 애플리케이션 레벨에서 처리

### 7.3 날짜/시간 표현

- ISO8601 형식 문자열 (`2026-03-25T10:00:00Z`) 권장
- TTL은 Unix epoch (초) 정수 사용

### 7.4 배치 연산

- `BatchWriteItem`: 최대 25개 항목
- `BatchGetItem`: 최대 100개 항목
- RCU/WCU 모니터링 필수

---

## 8.Capacity 설정 가이드

### 8.1 권장 설정

| 테이블 | 모드 | 초기 RCU/WCU |
|--------|------|-------------|
| VehicleLatestLocation | On-Demand | - |
| IngestDedupGuard | On-Demand | - |
| SessionStore | Provisioned | 10/10 |
| DeviceCertificateMetadata | On-Demand | - |

### 8.2 Auto Scaling 정책

```javascript
{
  "TargetTrackingScalingPolicyConfiguration": {
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "DynamoDBReadCapacityUtilization"
    }
  }
}
```

---

## 9. 모니터링/알람

### 9.1 CloudWatch Metrics

| 지표 | 임계값 | 알람 |
|------|--------|------|
| ConsumedReadCapacityUnits | > 80% | WARNING |
| ConsumedWriteCapacityUnits | > 80% | WARNING |
| ThrottledRequests | > 0 | CRITICAL |
| UserErrors | > 0 | WARNING |
| SystemErrors | > 0 | CRITICAL |

### 9.2 DynamoDB Streams Lag 모니터링

- Kinesis Data Analytics 또는 Lambda 처리 지연 모니터링
- P99 지연시간 < 5초 목표

---

## 10. 변경 이력 (Changelog)

- **v1.1 (2026-03-25):**
  - AWS 리전 `ap-northeast-2` (서울) 명시

- **v1.0 (2026-03-25):**
  - 최초 물리 스키마 명세서 작성
  - TrackCar 플랫폼 아키텍처 상세 설계 v2.4.0 기준 적용
  - 테이블 4개 정의 (VehicleLatestLocation, IngestDedupGuard, SessionStore, DeviceCertificateMetadata)
  - GSI 6개 정의
  - TTL 정책 정의

---

## 11. 참고 문서

| 문서명 | 버전 | 요약 |
|--------|------|------|
| TrackCar 플랫폼 아키텍처 상세 설계 | v2.4.0 | 전체 시스템 아키텍처 |
| TrackCar Ingest 검증 명세서 | v2.2 | 수집 파이프라인 검증 로직 |
| TrackCar 데이터 보관파기 정책 | v1.0 | TTL 정책 기준 |
| AWS IoT Core Developer Guide | - | JITR 인증서 관리 |
