# TrackCar Mobile App API Specification

- 작성일: 2026-03-25
- 버전: v1.0
- 상태: 초안

---

## 1. 문서 개요

### 1.1 목적

본 문서는 TrackCar Mobile App의 REST API를 정의한다. 각 API의 엔드포인트, 요청/응답 스키마, 오류 코드, 인증 및 권한 정책을 명시하여 프론트엔드 개발자와 백엔드 개발자 간 계약(Contract)을 명확히 한다.

### 1.2 기본 규칙

- **Base URL**: `/v1/mobile`
- **Content-Type**: `application/json`
- **Character Encoding**: UTF-8
- **Authentication**: AWS Cognito JWT Bearer Token
- **Authorization**: Role-based (OWNER, STAFF)

### 1.3 AWS 연동 설정

#### API Gateway

```
Base URL: https://qd1bmh3rdf.execute-api.ap-northeast-2.amazonaws.com/v1/mobile
Region: ap-northeast-2 (서울)
```

#### Cognito User Pool

```
User Pool ID: (Terraform outputs에서 확인)
Client ID: (Terraform outputs에서 확인)
Region: ap-northeast-2
```

#### 환경 변수 (.env)

```env
NEXT_PUBLIC_API_BASE_URL=https://qd1bmh3rdf.execute-api.ap-northeast-2.amazonaws.com/v1/mobile
NEXT_PUBLIC_COGNITO_USER_POOL_ID=ap-northeast-2_xxxxxx
NEXT_PUBLIC_COGNITO_CLIENT_ID=xxxxxxxxxxxxx
NEXT_PUBLIC_COGNITO_REGION=ap-northeast-2
```

### 1.4 인증 플로우

1. **로그인**: Cognito User Pool에서 username/password 인증
2. **토큰 수령**: ID Token, Access Token, Refresh Token 반환
3. **API 호출**: `Authorization: Bearer {idToken}` 헤더 포함
4. **토큰 갱신**: Access Token 만료 시 Refresh Token으로 갱신

#### 요청 예시

```http
GET /v1/mobile/vehicles HTTP/1.1
Host: qd1bmh3rdf.execute-api.ap-northeast-2.amazonaws.com
Authorization: Bearer eyJraWQiOiJ...
Content-Type: application/json
```

#### 토큰 갱신 예시

```typescript
// Refresh Token으로 새 토큰 요청
const response = await fetch(`https://trackcar.auth.ap-northeast-2.amazoncognito.com/oauth2/token`, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded',
  },
  body: new URLSearchParams({
    grant_type: 'refresh_token',
    client_id: process.env.NEXT_PUBLIC_COGNITO_CLIENT_ID,
    refresh_token: localStorage.getItem('refreshToken'),
  }),
});
```

### 1.3 공통 응답 형식

모든 API 응답은 다음 공통 형식을 따른다:

```json
{
  "success": true,
  "data": { ... },
  "meta": {
    "timestamp": "2026-03-25T10:30:00Z",
    "requestId": "req_abc123"
  }
}
```

**오류 응답:**
```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "사용자 친화적 오류 메시지",
    "details": { ... }
  },
  "meta": {
    "timestamp": "2026-03-25T10:30:00Z",
    "requestId": "req_abc123"
  }
}
```

### 1.4 공통 오류 코드

| HTTP Status | Error Code | 설명 |
|-------------|------------|------|
| 400 | `VALIDATION_ERROR` | 요청 파라미터 검증 실패 |
| 400 | `INVALID_REQUEST` | 잘못된 요청 형식 |
| 401 | `UNAUTHORIZED` | 인증 실패 |
| 403 | `FORBIDDEN` | 권한 없음 |
| 404 | `NOT_FOUND` | 리소스不存在 |
| 409 | `CONFLICT` | 리소스 충돌 (중복 등) |
| 500 | `INTERNAL_ERROR` | 서버 내부 오류 |

---

## 2. Dashboard API

### 2.1 대시보드 조회

**Endpoint:** `GET /v1/mobile/dashboard`

**목적:** 모바일 홈 대시보드 조회 - KPI, 조치 필요 차량, 최근 알림 요약

**권한:** OWNER, STAFF

**Request Query Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| organization_id | string | Yes | 조직 ID |
| group_id | string | No | 그룹 ID |

**Request Example:**
```
GET /v1/mobile/dashboard?organization_id=org_001&group_id=grp_001
```

**Response:**
```json
{
  "success": true,
  "data": {
    "dashboard": {
      "last_refresh_at": "2026-03-25T10:30:00Z",
      "assigned_vehicle_count": 45,
      "unassigned_vehicle_count": 3,
      "running_vehicle_count": 12,
      "offline_vehicle_count": 5,
      "high_alert_count": 2,
      "risk_vehicles": [
        {
          "vehicle_id": "veh_001",
          "vehicle_no": "12가3456",
          "status": "OFFLINE",
          "driver_name": "홍길동",
          "group_name": "서울지점",
          "last_received_at": "2026-03-25T09:00:00Z"
        }
      ],
      "recent_alerts": [
        {
          "alert_id": "alt_001",
          "severity": "HIGH",
          "title": "장치 오프라인",
          "vehicle_no": "12가3456",
          "occurred_at": "2026-03-25T10:00:00Z",
          "status": "UNREAD"
        }
      ]
    }
  }
}
```

---

## 3. Vehicle API

### 3.1 차량 목록 조회

**Endpoint:** `GET /v1/mobile/vehicles`

**목적:** 차량 목록 조회 - 검색, 필터, 정렬, 페이지네이션

**권한:** OWNER, STAFF

**Request Query Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| organization_id | string | Yes | 조직 ID |
| keyword | string | No | 차량번호/별칭/기사명 검색 |
| group_id | string | No | 그룹 ID |
| vehicle_status | string | No | REGISTERED, DEVICE_LINKED, DRIVER_LINKED, VERIFIED, ACTIVE, INACTIVE |
| trip_status | string | No | RUNNING, IDLE, OFFLINE, WARNING |
| install_status | string | No | NOT_INSTALLED, INSTALLED, VERIFIED, FAILED, REPLACED |
| assigned_status | string | No | ASSIGNED, UNASSIGNED |
| page | number | No | 페이지 번호 (기본: 1) |
| size | number | No | 페이지당 건수 (기본: 20, 최대: 50) |
| sort | string | No | 정렬 (last_received_at.desc, vehicle_no.asc, updated_at.desc) |

**Request Example:**
```
GET /v1/mobile/vehicles?organization_id=org_001&vehicle_status=ACTIVE&page=1&size=20
```

**Response:**
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "vehicle_id": "veh_001",
        "vehicle_no": "12가3456",
        "vehicle_type": "TRUCK",
        "alias": "1번차",
        "group_name": "서울지점",
        "vehicle_status": "VERIFIED",
        "trip_status": "RUNNING",
        "install_status": "VERIFIED",
        "assigned_status": "ASSIGNED",
        "driver_name": "홍길동",
        "last_received_at": "2026-03-25T10:00:00Z",
        "location_summary": "서울 강남구"
      }
    ],
    "page_info": {
      "page": 1,
      "size": 20,
      "total": 48,
      "has_next": true
    }
  }
}
```

### 3.2 차량 상세 조회

**Endpoint:** `GET /v1/mobile/vehicles/{vehicleId}`

**목적:** 차량 상세 정보 조회 - 상태, 위치, 기사 연결, 알림, 운행 요약

**권한:** OWNER, STAFF

**Path Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| vehicleId | string | Yes | 차량 ID |

**Request Query Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| organization_id | string | Yes | 조직 ID |

**Response:**
```json
{
  "success": true,
  "data": {
    "vehicle": {
      "vehicle_id": "veh_001",
      "vehicle_no": "12가3456",
      "vehicle_type": "TRUCK",
      "vin": "1HGBH41JXMN109186",
      "alias": "1번차",
      "group_id": "grp_001",
      "group_name": "서울지점",
      "vehicle_status": "VERIFIED",
      "trip_status": "RUNNING",
      "install_status": "VERIFIED",
      "assigned_status": "ASSIGNED",
      "driver_link": {
        "driver_id": "drv_001",
        "driver_name": "홍길동",
        "phone": "01012345678",
        "driver_code": "DRV20260001",
        "connected": true
      },
      "current_location": {
        "latitude": 37.5665,
        "longitude": 126.9780,
        "address": "서울 강남구",
        "received_at": "2026-03-25T10:00:00Z"
      },
      "last_received_at": "2026-03-25T10:00:00Z"
    },
    "recent_alerts": [
      {
        "alert_id": "alt_001",
        "severity": "HIGH",
        "title": "장치 오프라인",
        "occurred_at": "2026-03-25T10:00:00Z",
        "status": "UNREAD"
      }
    ],
    "recent_trips": [
      {
        "trip_id": "trip_001",
        "started_at": "2026-03-25T08:00:00Z",
        "ended_at": "2026-03-25T10:00:00Z",
        "distance_km": 45.2,
        "duration_min": 120
      }
    ]
  }
}
```

### 3.3 담당 기사 변경

**Endpoint:** `PATCH /v1/mobile/vehicles/{vehicleId}/driver-link`

**목적:** 차량-기사 연결 변경

**권한:** OWNER

**Path Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| vehicleId | string | Yes | 차량 ID |

**Request Body:**
```json
{
  "driver_id": "drv_001"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "vehicle_id": "veh_001",
    "driver_id": "drv_001",
    "updated": true
  }
}
```

### 3.4 그룹 변경

**Endpoint:** `PATCH /v1/mobile/vehicles/{vehicleId}/group`

**목적:** 차량 소속 그룹 변경

**권한:** OWNER

**Path Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| vehicleId | string | Yes | 차량 ID |

**Request Body:**
```json
{
  "group_id": "grp_002"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "vehicle_id": "veh_001",
    "group_id": "grp_002",
    "updated": true
  }
}
```

---

## 4. Trip API

### 4.1 운행 이력 목록 조회

**Endpoint:** `GET /v1/mobile/trips`

**목적:** 운행 이력 목록 조회 - 기간, 차량, 그룹 기준

**권한:** OWNER, STAFF

**Request Query Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| organization_id | string | Yes | 조직 ID |
| from_date | string | Yes | 조회 시작일 (YYYY-MM-DD) |
| to_date | string | Yes | 조회 종료일 (YYYY-MM-DD, 최대 93일) |
| vehicle_id | string | No | 차량 ID |
| group_id | string | No | 그룹 ID |
| trip_status | string | No | RUNNING, COMPLETED, CANCELLED |
| page | number | No | 페이지 번호 |
| size | number | No | 페이지당 건수 |

**Request Example:**
```
GET /v1/mobile/trips?organization_id=org_001&from_date=2026-03-18&to_date=2026-03-25
```

**Response:**
```json
{
  "success": true,
  "data": {
    "summary": {
      "trip_count": 125,
      "total_distance_km": 3250.5,
      "total_duration_min": 18750
    },
    "items": [
      {
        "trip_id": "trip_001",
        "vehicle_id": "veh_001",
        "vehicle_no": "12가3456",
        "driver_name": "홍길동",
        "started_at": "2026-03-25T08:00:00Z",
        "ended_at": "2026-03-25T10:00:00Z",
        "distance_km": 45.2,
        "duration_min": 120,
        "status": "COMPLETED"
      }
    ],
    "page_info": {
      "page": 1,
      "size": 20,
      "total": 125,
      "has_next": true
    }
  }
}
```

### 4.2 운행 상세 조회

**Endpoint:** `GET /v1/mobile/trips/{tripId}`

**목적:** 개별 운행 이력의 경로, 시간, 거리, 이벤트 조회

**권한:** OWNER, STAFF

**Path Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| tripId | string | Yes | 운행 ID |

**Request Query Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| organization_id | string | Yes | 조직 ID |

**Response:**
```json
{
  "success": true,
  "data": {
    "summary": {
      "trip_id": "trip_001",
      "vehicle_id": "veh_001",
      "vehicle_no": "12가3456",
      "driver_name": "홍길동",
      "started_at": "2026-03-25T08:00:00Z",
      "ended_at": "2026-03-25T10:00:00Z",
      "distance_km": 45.2,
      "duration_min": 120,
      "status": "COMPLETED"
    },
    "route_points": [
      {
        "latitude": 37.5665,
        "longitude": 126.9780,
        "recorded_at": "2026-03-25T08:00:00Z"
      },
      {
        "latitude": 37.5745,
        "longitude": 126.9850,
        "recorded_at": "2026-03-25T09:00:00Z"
      }
    ],
    "events": [
      {
        "event_type": "IGNITION_ON",
        "description": "시동 ON",
        "event_time": "2026-03-25T08:00:00Z",
        "location_summary": "서울 강남구"
      },
      {
        "event_type": "STOP",
        "description": "정차",
        "event_time": "2026-03-25T08:30:00Z",
        "duration_min": 10,
        "location_summary": "서울 서초구"
      },
      {
        "event_type": "IGNITION_OFF",
        "description": "시동 OFF",
        "event_time": "2026-03-25T10:00:00Z",
        "location_summary": "서울 송파구"
      }
    ]
  }
}
```

---

## 5. Alert API

### 5.1 알림 목록 조회

**Endpoint:** `GET /v1/mobile/alerts`

**목적:** 알림 목록 조회 - 심각도, 상태, 차량, 그룹 필터

**권한:** OWNER, STAFF

**Request Query Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| organization_id | string | Yes | 조직 ID |
| severity | string | No | CRITICAL, HIGH, MEDIUM, LOW |
| status | string | No | UNREAD, READ |
| vehicle_id | string | No | 차량 ID |
| group_id | string | No | 그룹 ID |
| page | number | No | 페이지 번호 |
| size | number | No | 페이지당 건수 |

**Response:**
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "alert_id": "alt_001",
        "severity": "HIGH",
        "title": "장치 오프라인",
        "description": "장치 통신이 30분 이상 중단되었습니다.",
        "vehicle_id": "veh_001",
        "vehicle_no": "12가3456",
        "occurred_at": "2026-03-25T10:00:00Z",
        "status": "UNREAD"
      }
    ],
    "total_unread": 5,
    "page_info": {
      "page": 1,
      "size": 20,
      "total": 25,
      "has_next": true
    }
  }
}
```

### 5.2 알림 일괄 확인 처리

**Endpoint:** `PATCH /v1/mobile/alerts/read-all`

**목적:** 현재 필터 조건의 미확인 알림 일괄 확인 처리

**권한:** OWNER

**Request Body:**
```json
{
  "organization_id": "org_001",
  "severity": "HIGH",
  "status": "UNREAD"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "updated_count": 3
  }
}
```

### 5.3 알림 상세 조회

**Endpoint:** `GET /v1/mobile/alerts/{alertId}`

**목적:** 알림 상세 조회 - 원인, 상태, 차량 정보, 조치 힌트

**권한:** OWNER, STAFF

**Path Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| alertId | string | Yes | 알림 ID |

**Response:**
```json
{
  "success": true,
  "data": {
    "alert": {
      "alert_id": "alt_001",
      "severity": "HIGH",
      "title": "장치 오프라인",
      "status": "UNREAD",
      "description": "장치 통신이 30분 이상 중단되었습니다.",
      "occurred_at": "2026-03-25T10:00:00Z",
      "vehicle": {
        "vehicle_id": "veh_001",
        "vehicle_no": "12가3456",
        "status": "OFFLINE",
        "last_received_at": "2026-03-25T09:30:00Z"
      }
    }
  }
}
```

### 5.4 알림 확인 처리

**Endpoint:** `PATCH /v1/mobile/alerts/{alertId}/read`

**목적:** 알림 확인 완료 처리

**권한:** OWNER, STAFF

**Path Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| alertId | string | Yes | 알림 ID |

**Response:**
```json
{
  "success": true,
  "data": {
    "alert_id": "alt_001",
    "status": "READ",
    "updated": true
  }
}
```

---

## 6. Workspace API

### 6.1 워크스페이스 요약 조회

**Endpoint:** `GET /v1/mobile/workspace`

**목적:** 워크스페이스 요약 - 담당자, 그룹, 연결 현황 요약

**권한:** OWNER

**Request Query Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| organization_id | string | Yes | 조직 ID |

**Response:**
```json
{
  "success": true,
  "data": {
    "staff_summary": {
      "total_count": 5,
      "active_count": 4,
      "inactive_count": 1
    },
    "group_summary": {
      "total_count": 3,
      "group_names": ["서울지점", "부산지점", "대전지점"]
    },
    "link_summary": {
      "total_vehicles": 48,
      "assigned_count": 45,
      "unassigned_count": 3
    }
  }
}
```

---

## 7. Staff User API

### 7.1 담당자 목록 조회

**Endpoint:** `GET /v1/mobile/staff-users`

**목적:** 담당자 계정 목록 조회

**권한:** OWNER

**Request Query Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| organization_id | string | Yes | 조직 ID |
| keyword | string | No | 이름/이메일 검색 |
| status | string | No | ACTIVE, INACTIVE, ALL |
| page | number | No | 페이지 번호 |
| size | number | No | 페이지당 건수 |

**Response:**
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "staff_user_id": "staff_001",
        "name": "김철수",
        "email": "kim@company.co.kr",
        "status": "ACTIVE",
        "notification_channels": ["PUSH", "EMAIL"],
        "scope_type": "ALL_GROUPS",
        "group_ids": [],
        "created_at": "2026-03-01T09:00:00Z"
      }
    ],
    "page_info": {
      "page": 1,
      "size": 20,
      "total": 5,
      "has_next": false
    }
  }
}
```

### 7.2 담당자 생성

**Endpoint:** `POST /v1/mobile/staff-users`

**목적:** 신규 담당자 계정 생성

**권한:** OWNER

**Request Body:**
```json
{
  "organization_id": "org_001",
  "name": "김철수",
  "email": "kim@company.co.kr",
  "phone": "01012345678",
  "notification_channels": ["PUSH", "EMAIL"],
  "scope_type": "SELECTED_GROUPS",
  "group_ids": ["grp_001", "grp_002"]
}
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| organization_id | string | Yes | 조직 ID |
| name | string | Yes | 담당자 이름 (2~30자) |
| email | string | Yes | 이메일 (중복 불가) |
| phone | string | Yes | 휴대폰번호 (KR 형식) |
| notification_channels | string[] | No | PUSH, EMAIL |
| scope_type | string | Yes | ALL_GROUPS, SELECTED_GROUPS |
| group_ids | string[] | Cond | scope_type=SELECTED_GROUPS 시 필수 |

**Response:**
```json
{
  "success": true,
  "data": {
    "staff_user_id": "staff_001",
    "invitation_status": "INVITED"
  }
}
```

### 7.3 담당자 상세 조회

**Endpoint:** `GET /v1/mobile/staff-users/{staffUserId}`

**목적:** 담당자 상세 정보 조회

**권한:** OWNER

**Path Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| staffUserId | string | Yes | 담당자 ID |

**Response:**
```json
{
  "success": true,
  "data": {
    "staff_user": {
      "staff_user_id": "staff_001",
      "name": "김철수",
      "email": "kim@company.co.kr",
      "phone": "01012345678",
      "status": "ACTIVE",
      "notification_channels": ["PUSH", "EMAIL"],
      "scope_type": "SELECTED_GROUPS",
      "group_ids": ["grp_001", "grp_002"],
      "created_at": "2026-03-01T09:00:00Z"
    }
  }
}
```

### 7.4 담당자 수정

**Endpoint:** `PATCH /v1/mobile/staff-users/{staffUserId}`

**목적:** 담당자 정보 수정

**권한:** OWNER

**Path Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| staffUserId | string | Yes | 담당자 ID |

**Request Body:**
```json
{
  "name": "김철수",
  "phone": "01098765432",
  "notification_channels": ["PUSH"],
  "scope_type": "ALL_GROUPS",
  "group_ids": []
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "staff_user_id": "staff_001",
    "updated": true
  }
}
```

### 7.5 담당자 상태 변경

**Endpoint:** `PATCH /v1/mobile/staff-users/{staffUserId}/status`

**목적:** 담당자 계정 활성화/비활성화

**권한:** OWNER

**Path Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| staffUserId | string | Yes | 담당자 ID |

**Request Body:**
```json
{
  "status": "INACTIVE"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "staff_user_id": "staff_001",
    "status": "INACTIVE",
    "updated": true
  }
}
```

---

## 8. Group API

### 8.1 그룹 목록 조회

**Endpoint:** `GET /v1/mobile/groups`

**목적:** 그룹 목록 조회

**권한:** OWNER

**Request Query Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| organization_id | string | Yes | 조직 ID |

**Response:**
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "group_id": "grp_001",
        "group_name": "서울지점",
        "description": "서울 지역 운송팀",
        "vehicle_count": 20,
        "staff_count": 3,
        "created_at": "2026-03-01T09:00:00Z"
      }
    ]
  }
}
```

### 8.2 그룹 생성

**Endpoint:** `POST /v1/mobile/groups`

**목적:** 그룹 생성

**권한:** OWNER

**Request Body:**
```json
{
  "organization_id": "org_001",
  "name": "부산지점",
  "description": "부산 지역 운송팀"
}
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| organization_id | string | Yes | 조직 ID |
| name | string | Yes | 그룹명 (2~30자, 조직 내 중복 불가) |
| description | string | No | 그룹 설명 (최대 200자) |

**Response:**
```json
{
  "success": true,
  "data": {
    "group_id": "grp_002"
  }
}
```

### 8.3 그룹 수정

**Endpoint:** `PATCH /v1/mobile/groups/{groupId}`

**목적:** 그룹 정보 수정

**권한:** OWNER

**Path Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| groupId | string | Yes | 그룹 ID |

**Request Body:**
```json
{
  "name": "부산지점",
  "description": "부산 지역 운송팀 수정"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "group_id": "grp_002",
    "updated": true
  }
}
```

### 8.4 그룹 삭제

**Endpoint:** `DELETE /v1/mobile/groups/{groupId}`

**목적:** 그룹 삭제 (소속 차량이 없을 때만 가능)

**권한:** OWNER

**Path Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| groupId | string | Yes | 그룹 ID |

**Response:**
```json
{
  "success": true,
  "data": {
    "group_id": "grp_002",
    "deleted": true
  }
}
```

---

## 9. Vehicle-Driver Link API

### 9.1 차량-기사 연결 목록 조회

**Endpoint:** `GET /v1/mobile/vehicle-driver-links`

**목적:** 차량-기사 상시 연결 현황 조회

**권한:** OWNER

**Request Query Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| organization_id | string | Yes | 조직 ID |
| group_id | string | No | 그룹 ID |
| assigned_status | string | No | ASSIGNED, UNASSIGNED |
| keyword | string | No | 차량번호/기사명 검색 |
| page | number | No | 페이지 번호 |
| size | number | No | 페이지당 건수 |

**Response:**
```json
{
  "success": true,
  "data": {
    "summary": {
      "unassigned_vehicle_count": 3
    },
    "items": [
      {
        "vehicle_id": "veh_001",
        "vehicle_no": "12가3456",
        "group_name": "서울지점",
        "driver_id": "drv_001",
        "driver_name": "홍길동",
        "assigned": true,
        "assigned_at": "2026-03-01T09:00:00Z"
      },
      {
        "vehicle_id": "veh_002",
        "vehicle_no": "34나5678",
        "group_name": "서울지점",
        "driver_id": null,
        "driver_name": null,
        "assigned": false,
        "assigned_at": null
      }
    ],
    "page_info": {
      "page": 1,
      "size": 20,
      "total": 48,
      "has_next": true
    }
  }
}
```

### 9.2 차량-기사 연결 변경

**Endpoint:** `PUT /v1/mobile/vehicle-driver-links/{vehicleId}`

**목적:** 차량-기사 연결 생성 또는 변경

**권한:** OWNER

**Path Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| vehicleId | string | Yes | 차량 ID |

**Request Body:**
```json
{
  "driver_id": "drv_001"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "vehicle_id": "veh_001",
    "driver_id": "drv_001",
    "linked": true
  }
}
```

### 9.3 차량-기사 연결 해제

**Endpoint:** `DELETE /v1/mobile/vehicle-driver-links/{vehicleId}`

**목적:** 차량-기사 연결 해제

**권한:** OWNER

**Path Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| vehicleId | string | Yes | 차량 ID |

**Request Body:**
```json
{
  "vehicle_id": "veh_001"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "vehicle_id": "veh_001",
    "unlinked": true
  }
}
```

---

## 10. My Account API

### 10.1 내 정보 조회

**Endpoint:** `GET /v1/mobile/me`

**목적:** 내 계정 정보, 알림 설정, 조직 정보 조회

**권한:** OWNER, STAFF

**Response:**
```json
{
  "success": true,
  "data": {
    "me": {
      "user_id": "usr_001",
      "name": "김철수",
      "email": "kim@company.co.kr",
      "role": "OWNER",
      "organization": {
        "organization_id": "org_001",
        "name": "테스트운수",
        "default_group_name": "기본그룹"
      },
      "notification_settings": {
        "push_enabled": true,
        "email_enabled": true
      }
    }
  }
}
```

### 10.2 알림 설정 변경

**Endpoint:** `PATCH /v1/mobile/me/notification-settings`

**목적:** 푸시/이메일 알림 설정 변경

**권한:** OWNER, STAFF

**Request Body:**
```json
{
  "push_enabled": true,
  "email_enabled": false
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "notification_settings": {
      "push_enabled": true,
      "email_enabled": false
    },
    "updated": true
  }
}
```

---

## 11. 공통 타입 정의

### 11.1 Status Enum Values

| Enum | 값 |
|------|-----|
| **vehicle_status** | REGISTERED, DEVICE_LINKED, DRIVER_LINKED, VERIFIED, ACTIVE, INACTIVE |
| **trip_status** | RUNNING, IDLE, OFFLINE, WARNING, COMPLETED, CANCELLED |
| **install_status** | NOT_INSTALLED, INSTALLED, VERIFIED, FAILED, REPLACED |
| **alert_severity** | CRITICAL, HIGH, MEDIUM, LOW |
| **alert_status** | UNREAD, READ |
| **user_role** | OWNER, STAFF |
| **staff_status** | ACTIVE, INACTIVE |
| **scope_type** | ALL_GROUPS, SELECTED_GROUPS |
| **notification_channel** | PUSH, EMAIL |
| **assigned_status** | ASSIGNED, UNASSIGNED |

### 11.2 Page Info

```json
{
  "page": 1,
  "size": 20,
  "total": 100,
  "has_next": true
}
```

---

## 12. 변경 이력 (Changelog)

- **v1.0 (2026-03-25):**
  - 최초 API Spec 문서 작성
  - Mobile App 화면 상세 설계서 기반 API 정의
  - Dashboard, Vehicle, Trip, Alert, Workspace, Staff User, Group, Vehicle-Driver Link, My Account API 포함

---

## 13. 참고 문서

| 문서명 | 설명 |
|--------|------|
| TrackCar Mobile App 요구사항 및 기능 명세서 | 기능/요구사항 정의 |
| TrackCar Mobile App 화면 상세 설계서 | 화면별 상세 설계 |
| TrackCar Mobile App 프로세스 및 IA 설계서 | 프로세스/메뉴 구조 |
| TrackCar 플랫폼 아키텍처 상세 설계 | Backend 아키텍처 |
| TrackCar Installer Admin Web API Specification | Installer Admin API Spec |
