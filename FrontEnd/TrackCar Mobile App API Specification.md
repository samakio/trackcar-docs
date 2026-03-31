# TrackCar Mobile App API Specification

- 작성일: 2026-03-31
- 버전: v2.0
- 상태: 재정비 초안

---

## 1. 문서 목적

본 문서는 TrackCar Mobile App의 API 계약을 정리한다.

주의:
- 본 문서는 **모바일 MVP 설계 기준**이다.
- 일부 Owner 운영 기능은 backend 보강이 필요한 **계획 API**를 포함한다.
- 현재 구현과 계획 범위를 함께 표기한다.

---

## 2. 공통 규칙

- Base URL: `/v1/mobile`
- 인증: AWS Cognito JWT Bearer Token
- 역할: `OWNER`, `STAFF`
- 응답 envelope: `success / data / error / meta`

### 공통 응답 예시

```json
{
  "success": true,
  "data": {},
  "meta": {
    "timestamp": "2026-03-31T10:00:00Z",
    "requestId": "req_abc123"
  }
}
```

### 공통 오류 예시

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "오류 메시지"
  },
  "meta": {
    "timestamp": "2026-03-31T10:00:00Z",
    "requestId": "req_abc123"
  }
}
```

---

## 3. API 범위 요약

| 영역 | Endpoint | 상태 |
|------|----------|------|
| Home | `GET /dashboard` | 구현 |
| Vehicles | `GET /vehicles` | 구현 |
| Vehicles | `GET /vehicles/{vehicleId}` | 구현 |
| Vehicle Map | `GET /vehicles/map` | 구현 |
| Trips | `GET /trips` | 부분 구현 |
| Trips | `GET /trips/{tripId}` | 부분 구현 |
| Alerts | `GET /alerts` | 부분 구현 |
| Alerts | `GET /alerts/{alertId}` | 계획/드리프트 존재 |
| Alerts | `PATCH /alerts/{alertId}/read` | 구현 |
| Alerts | `PATCH /alerts/read-all` | 구현 |
| My | `GET /me` | 구현 |
| My | `PATCH /me/notification-settings` | 구현 |
| Operations | `GET/POST/PATCH /staff-users...` | 구현 |
| Operations | `GET/POST/PATCH/DELETE /groups...` | 구현 |
| Operations | `GET/PUT/DELETE /vehicle-driver-links...` | 부분 구현 |
| Operations | `GET/POST/PATCH /alert-settings...` | 구현 |

---

## 4. Home API

### 4.1 대시보드 조회

**Endpoint:** `GET /v1/mobile/dashboard`

**목적:** 모바일 홈 KPI 요약 조회

**권한:** OWNER, STAFF

**Request Query Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| organization_id | string | Yes | 조직 ID |
| group_id | string | No | 그룹 필터 |

**Target Response (설계 기준):**
```json
{
  "success": true,
  "data": {
    "dashboard": {
      "last_refresh_at": "2026-03-31T10:00:00Z",
      "total_vehicle_count": 120,
      "running_vehicle_count": 32,
      "idle_vehicle_count": 71,
      "offline_vehicle_count": 17
    }
  }
}
```

> 현재 backend는 KPI 4개(`total/running/idle/offline`)를 반환한다. 세부 계산 기준은 차량 `running_status`를 사용한다.

---

## 5. Vehicle API

### 5.1 차량 목록 조회

**Endpoint:** `GET /v1/mobile/vehicles`

**권한:** OWNER, STAFF

**Request Query Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| organization_id | string | Yes | 조직 ID |
| keyword | string | No | 차량번호/기사명 검색 |
| group_id | string | No | 그룹 ID |
| vehicle_status | string | No | 차량 상태 |
| trip_status | string | No | 운행 상태 |
| install_status | string | No | 설치 상태 |
| page | number | No | 페이지 번호 |
| size | number | No | 페이지 크기 |

**Response:**
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "vehicle_id": "veh_001",
        "vehicle_no": "경기45바6789",
        "vehicle_type": "TRUCK",
        "group_name": "통합테스트운수A 기본그룹",
        "vehicle_status": "VERIFIED",
        "trip_status": "RUNNING",
        "install_status": "VERIFIED",
        "driver_name": "홍길동A",
        "last_received_at": "2026-03-31T10:00:00Z",
        "location_summary": "서울 중구 세종대로"
      }
    ],
    "page_info": {
      "page": 1,
      "size": 20,
      "total": 1,
      "has_next": false
    }
  }
}
```

### 5.2 차량 상세 조회

**Endpoint:** `GET /v1/mobile/vehicles/{vehicleId}`

**권한:** OWNER, STAFF

**Response:**
```json
{
  "success": true,
  "data": {
    "vehicle": {
      "vehicle_id": "veh_001",
      "vehicle_no": "경기45바6789",
      "vehicle_status": "VERIFIED",
      "trip_status": "RUNNING",
      "install_status": "VERIFIED",
      "group_name": "통합테스트운수A 기본그룹",
      "driver_name": "홍길동A",
      "driver_code": "DRV-BIZ-A-001",
      "last_received_at": "2026-03-31T10:00:00Z",
      "current_location": {
        "latitude": 37.5665,
        "longitude": 126.9780,
        "address": "서울 중구 세종대로"
      }
    },
    "recent_alerts": [],
    "recent_trips": []
  }
}
```

### 5.3 전체차량위치 조회

**Endpoint:** `GET /v1/mobile/vehicles/map`

**권한:** OWNER, STAFF

**목적:** 전체 차량 위치 지도용 marker 데이터 조회

**Response:**
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "vehicle_id": "veh_001",
        "vehicle_no": "경기45바6789",
        "trip_status": "RUNNING",
        "latitude": 37.5665,
        "longitude": 126.9780,
        "last_received_at": "2026-03-31T10:00:00Z"
      }
    ]
  }
}
```

---

## 6. Trip API

### 6.1 운행 이력 목록 조회

**Endpoint:** `GET /v1/mobile/trips`

**권한:** OWNER, STAFF

**Request Query Parameters:**
- `organization_id`
- `from_date`
- `to_date`
- `vehicle_id`
- `group_id`
- `trip_status`
- `page`
- `size`

**Target Response:**
```json
{
  "success": true,
  "data": {
    "summary": {
      "trip_count": 12,
      "total_distance_km": 120.5,
      "total_duration_min": 340
    },
    "items": [],
    "page_info": {
      "page": 1,
      "size": 20,
      "total": 12,
      "has_next": false
    }
  }
}
```

### 6.2 운행 상세 조회

**Endpoint:** `GET /v1/mobile/trips/{tripId}`

**권한:** OWNER, STAFF

**Target Response:**
```json
{
  "success": true,
  "data": {
    "summary": {},
    "route_points": [],
    "events": []
  }
}
```

---

## 7. Alerts API

### 7.1 알림 목록 조회

**Endpoint:** `GET /v1/mobile/alerts`

**권한:** OWNER, STAFF

### 7.2 알림 상세 조회 `부분 구현`

**Endpoint:** `GET /v1/mobile/alerts/{alertId}`

### 7.3 알림 읽음 처리

**Endpoint:** `PATCH /v1/mobile/alerts/{alertId}/read`

### 7.4 알림 일괄 읽음 처리

**Endpoint:** `PATCH /v1/mobile/alerts/read-all`

> 현재 구현은 존재하지만 동작 검증과 범위 정리가 더 필요하다.

---

## 8. Operations API (OWNER)

### 8.1 담당자 관리

| Method | Endpoint | 설명 |
|--------|----------|------|
| GET | `/v1/mobile/staff-users` | 담당자 목록 |
| POST | `/v1/mobile/staff-users` | 담당자 생성 |
| PATCH | `/v1/mobile/staff-users/{staffUserId}` | 담당자 수정 |
| PATCH | `/v1/mobile/staff-users/{staffUserId}/status` | 활성/비활성 |

**주의사항:**
- 생성 시 Cognito 초대와 `app_user(role=STAFF)` 생성이 함께 수행된다.
- 수정 범위는 현재 `name`, `phone`, `group_id` 중심이다.
- 상태 변경은 `ACTIVE` / `INACTIVE` 기준이다.

### 8.2 그룹 관리

| Method | Endpoint | 설명 |
|--------|----------|------|
| GET | `/v1/mobile/groups` | 그룹 목록 |
| POST | `/v1/mobile/groups` | 그룹 생성 |
| PATCH | `/v1/mobile/groups/{groupId}` | 그룹 수정 |
| DELETE | `/v1/mobile/groups/{groupId}` | 그룹 삭제 |

### 8.3 알림 관리

| Method | Endpoint | 설명 |
|--------|----------|------|
| GET | `/v1/mobile/alert-settings` | 차량별 알림 설정 목록 |
| POST | `/v1/mobile/alert-settings` | 차량별 알림 설정 저장 |
| PATCH | `/v1/mobile/alert-settings/{settingId}` | 설정 수정 |

설정 범위:
- 시동 ON/OFF 알림
- 공회전 알림
- 위치 알림은 후속 범위

---

## 9. My API

### 9.1 내 정보 조회

**Endpoint:** `GET /v1/mobile/me`

**Response:**
```json
{
  "success": true,
  "data": {
    "me": {
      "user_id": "usr_001",
      "name": "홍길동",
      "email": "hong@test.com",
      "role": "OWNER",
      "organization": {
        "organization_id": "org_001",
        "name": "통합테스트운수A"
      },
      "notification_settings": {
        "push_enabled": true,
        "email_enabled": true
      }
    }
  }
}
```

### 9.2 내 알림 설정 변경

**Endpoint:** `PATCH /v1/mobile/me/notification-settings`

**Request Body:**
```json
{
  "push_enabled": true,
  "email_enabled": false
}
```

---

## 10. 구현 기준 안내

### 10.1 구현 우선순위
즉시 구현 권장:
- Home (축소 KPI)
- 전체차량위치
- Vehicles
- Vehicle Detail
- Trips
- Alerts
- My

후속 구현:
- 위치 알림

### 10.2 주의사항
- 현재 backend는 일부 모바일 API가 스텁 또는 부분 구현 상태다.
- 문서의 `계획` API는 모바일 설계 기준이며, backend 구현 동기화가 필요하다.
