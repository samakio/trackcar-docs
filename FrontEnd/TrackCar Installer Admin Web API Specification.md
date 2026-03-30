# TrackCar Installer Admin Web API Specification

- 작성일: 2026-03-25
- 버전: v1.5
- 상태: 초안

---

## 1. 문서 개요

### 1.1 목적

본 문서는 TrackCar Installer Admin Web의 REST API를 정의한다. 각 API의 엔드포인트, 요청/응답 스키마, 오류 코드, 인증 및 권한 정책을 명시하여 프론트엔드 개발자와 백엔드 개발자 간 계약(Contract)을 명확히 한다.

### 1.2 기본 규칙

- **Base URL**: `/api/admin`
- **Content-Type**: `application/json`
- **Character Encoding**: UTF-8
- **Authentication**: AWS Cognito JWT Bearer Token
- **Authorization**: Role-based (Installer, Operator, Admin)

### 1.3 AWS 연동 설정

#### API Gateway

```
Base URL: https://a1xpskzj48.execute-api.ap-northeast-2.amazonaws.com/v1/api/admin
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
VITE_API_BASE_URL=https://a1xpskzj48.execute-api.ap-northeast-2.amazonaws.com/v1/api/admin
VITE_COGNITO_USER_POOL_ID=ap-northeast-2_xxxxxx
VITE_COGNITO_CLIENT_ID=xxxxxxxxxxxxx
VITE_COGNITO_REGION=ap-northeast-2
```

### 1.4 인증 플로우

1. **로그인**: Cognito User Pool에서 username/password 인증
2. **토큰 수령**: ID Token, Access Token, Refresh Token 반환
3. **API 호출**: `Authorization: Bearer {idToken}` 헤더 포함
4. **토큰 갱신**: Access Token 만료 시 Refresh Token으로 갱신

#### 요청 예시

```http
GET /api/admin/owners HTTP/1.1
Host: a1xpskzj48.execute-api.ap-northeast-2.amazonaws.com
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
    client_id: import.meta.env.VITE_COGNITO_CLIENT_ID,
    refresh_token: localStorage.getItem('refreshToken'),
  }),
});
```

#### 최초 로그인 비밀번호 변경

- system user를 임시 비밀번호로 처음 로그인하면 Cognito가 `NEW_PASSWORD_REQUIRED`를 반환한다.
- 프론트는 새 비밀번호 입력 UI를 표시하고 `completeNewPasswordChallenge`를 수행한다.
- 변경 완료 후 정상 로그인 세션으로 전환된다.

### 1.5 현재 구현 범위 주의사항

- `members`는 owner에 소속된 하위 운영 계정 도메인이다.
- `members`는 `installer-admin-web` 로그인 대상이 아니며, `mobile app` 로그인 대상이다.
- `members`는 내부 운영 계정인 `system/users`와 다른 리소스이며, 운전자 계정인 `drivers`와도 구분된다.
- `members`의 앱 초대/활성화는 `App Activation API` 흐름과 연결된다.
- `group` 범위 연결은 현재 운영 스키마의 `user_group_access`를 전제로 한다.

### 1.6 공통 응답 형식

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
| 404 | `NOT_FOUND` | ресурс不存在 |
| 409 | `CONFLICT` | 리소스 충돌 (중복 등) |
| 500 | `INTERNAL_ERROR` | 서버 내부 오류 |

---

## 2. 대시보드 API

### 2.1 대시보드 조회

**Endpoint:** `GET /api/admin/dashboard`

**목적:** 설치 운영 현황 KPI, 차트 데이터, 이상 차량 목록 조회

**권한:** Installer, Operator, Admin

**Request Query Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| date_from | string | No | 조회 시작일 (YYYY-MM-DD) |
| date_to | string | No | 조회 종료일 (YYYY-MM-DD) |
| owner_type | string | No | OWNER 유형 (PERSONAL, BUSINESS, ALL) |
| group_id | string | No | 그룹 ID |
| device_model | string | No | DTG 모델명 |
| verification_status | string[] | No | 검증 상태 필터 |

**Request Example:**
```
GET /api/admin/dashboard?date_from=2026-03-01&date_to=2026-03-25&owner_type=ALL
```

**Response:**
```json
{
  "success": true,
  "data": {
    "metrics": {
      "installed_vehicle_count": 150,
      "online_device_count": 120,
      "unlinked_device_count": 10,
      "verification_pass_count": 140
    },
    "charts": {
      "install_trend": [
        {"date": "2026-03-01", "count": 145},
        {"date": "2026-03-02", "count": 147}
      ],
      "device_status": {
        "ONLINE": 100,
        "OFFLINE": 15,
        "NEVER_CONNECTED": 5,
        "ERROR": 5
      },
      "vehicle_driver_mapping": {
        "MAPPED": 130,
        "VEHICLE_ONLY": 10,
        "DRIVER_ONLY": 5,
        "UNMAPPED": 5
      },
      "activation_funnel": [
        {"step": "장치 등록", "count": 150, "percent": 100},
        {"step": "차량 매핑", "count": 130, "percent": 87},
        {"step": "기사 지정", "count": 118, "percent": 79},
        {"step": "서비스 활성화", "count": 102, "percent": 68}
      ]
    },
    "lists": {
      "recent_issue_vehicles": [
        {
          "vehicle_id": "veh_001",
          "vehicle_no": "12가3456",
          "device_serial_no": "DTG-001",
          "vehicle_status": "DEVICE_LINKED",
          "installation_status": "VERIFIED",
          "severity": "HIGH",
          "last_received_at": "2026-03-25T10:00:00Z",
          "location_summary": "서울 강남구"
        }
      ]
    }
  },
  "meta": {
    "timestamp": "2026-03-25T10:30:00Z",
    "requestId": "req_dash_001"
  }
}
```

---

## 3. Owner API

### 3.1 Owner 목록 조회

**Endpoint:** `GET /api/admin/owners`

**목적:** Owner 계정 목록 조회

**권한:** Installer, Operator, Admin

**Request Query Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| keyword | string | No | 이름/회사명/휴대폰/사업자번호 검색 |
| owner_type | string | No | PERSONAL, BUSINESS, ALL |
| status | string | No | ACTIVE, INACTIVE, LOCKED, ALL |
| group_id | string | No | 그룹 ID |
| page | number | No | 페이지 번호 (기본: 1) |
| size | number | No | 페이지당 건수 (기본: 20, 최대: 100) |
| sort | string | No | 정렬 (created_at.desc, created_at.asc, name.asc, name.desc) |

**Response:**
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "owner_id": "7797a28f-c78c-43c8-8bbe-4d69cbdf8b2a",
        "owner_type": "BUSINESS",
        "name_or_company_name": "테스트운수",
        "phone": "01012345678",
        "business_no": "1234567890",
        "transport_biz_no": "운송12345",
        "default_group_name": "기본그룹",
        "vehicle_count": 10,
        "driver_count": 15,
        "unmapped_vehicle_count": 3,
        "app_activation_status": "ACTIVE",
        "created_at": "2026-03-01T09:00:00Z"
      }
    ],
    "page_info": {
      "page": 1,
      "size": 20,
      "total": 45,
      "has_next": true
    }
  }
}
```

### 3.2 Owner 상세 조회

**Endpoint:** `GET /api/admin/owners/{ownerId}`

**목적:** Owner 상세 정보 조회

**권한:** Installer, Operator, Admin

**Path Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| ownerId | string | Yes | Owner ID |

**Response:**
```json
{
  "success": true,
  "data": {
    "owner": {
      "owner_id": "own_001",
      "owner_type": "BUSINESS",
      "name_or_company_name": "테스트운수",
      "phone": "01012345678",
      "email": "test@test.co.kr",
      "business_no": "1234567890",
      "transport_biz_no": "운송12345",
      "default_group_name": "기본그룹",
      "vehicle_count": 10,
      "driver_count": 15,
      "unmapped_vehicle_count": 3,
      "status": "ACTIVE",
      "created_at": "2026-03-01T09:00:00Z",
      "updated_at": "2026-03-25T10:00:00Z"
    }
  }
}
```

### 3.3 Owner 생성

**Endpoint:** `POST /api/admin/owners`

**목적:** 신규 고객사 등록 (그룹/OWNER앱계정/기사 자동 생성 포함)

**권한:** Installer, Operator, Admin

**자동 생성 항목:**
| owner_type | 기본 그룹 | OWNER 앱 계정 | 기사 |
|-----------|:---------:|:------------:|:----:|
| PERSONAL | ✅ 자동 | ✅ 자동 | ✅ 자동 |
| BUSINESS | ✅ 자동 | ✅ 자동 | ❌ |

- OWNER 앱 계정은 DB만 생성 (Cognito 초대는 고객사 상세에서 별도 진행)
- PERSONAL 기사: `driver_code = DRV-{ownerId앞8자리}`, 전화번호/이름은 고객사와 동일

**Request Body:**
```json
{
  "owner_type": "BUSINESS",
  "name_or_company_name": "테스트운수",
  "phone": "01012345678",
  "email": "test@test.co.kr",
  "business_no": "1234567890",
  "transport_biz_no": "운송12345",
  "default_group_name": "테스트운수 기본그룹"
}
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| owner_type | string | Yes | PERSONAL, BUSINESS |
| name_or_company_name | string | Yes | 이름 또는 회사명 (2~100자) |
| phone | string | Yes | 휴대폰번호 (KR 형식) |
| email | string | **Yes** | 이메일 (OWNER 앱 계정 생성에 필요) |
| business_no | string | No | 사업자번호 |
| transport_biz_no | string | No | 운송사업자등록번호 (eTAS 전송용) |
| default_group_name | string | No | 기본 그룹명 (미입력 시 "{이름} 기본그룹") |

**Response (BUSINESS):**
```json
{
  "success": true,
  "data": {
    "owner_id": "own_001",
    "default_group_id": "grp_001",
    "owner_account_id": "usr_001"
  }
}
```

**Response (PERSONAL):**
```json
{
  "success": true,
  "data": {
    "owner_id": "own_001",
    "default_group_id": "grp_001",
    "owner_account_id": "usr_001",
    "driver_id": "drv_001"
  }
}
```

### 3.4 고객사 수정

**Endpoint:** `PUT /api/admin/owners/{ownerId}`

**목적:** 고객사 정보 수정

**권한:** Operator, Admin

**수정 가능 필드:** `name_or_company_name`, `phone`, `email`, `business_no`, `transport_biz_no`, `status`

**수정 불가 필드 (읽기 전용):** `owner_type` — 생성 시 결정되며 이후 변경 불가

**주의사항:**
- `phone`, `business_no` 변경 시 중복 여부를 `POST /api/admin/owners/check-duplicate`로 먼저 확인 권장
- `status` 변경은 이 API에 포함됨 (별도 상태 변경 API 없음)

**Request Body:**
```json
{
  "name_or_company_name": "수정회사명",
  "phone": "01098765432",
  "email": "updated@test.co.kr",
  "business_no": "0987654321",
  "transport_biz_no": "운송54321",
  "status": "ACTIVE"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "owner_id": "own_001"
  }
}
```

### 3.5 Owner 중복 확인

**Endpoint:** `POST /api/admin/owners/check-duplicate`

**목적:** 휴대폰번호/사업자번호 중복 확인

**권한:** Installer, Operator, Admin

**Request Body:**
```json
{
  "owner_type": "BUSINESS",
  "phone": "01012345678",
  "business_no": "1234567890"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "duplicated": false,
    "duplicated_fields": []
  }
}
```

### 3.6 고객사 삭제

**Endpoint:** `DELETE /api/admin/owners/{ownerId}`

**목적:** 고객사 삭제 (soft delete)

**권한:** Operator, Admin

**Path Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| ownerId | string | Yes | 고객사 ID |

**비즈니스 조건:** 소속 그룹, 차량, 기사, 회원 중 하나라도 존재하면 400 오류 반환

**Response (성공):**
```json
{
  "success": true,
  "data": null
}
```

**Response (삭제 불가):**
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Cannot delete owner with groups, vehicles, drivers, or members"
  }
}
```

### 3.7 고객사 앱 계정 조회

**Endpoint:** `GET /api/admin/owners/{ownerId}/account`

**목적:** 해당 고객사의 OWNER 앱 계정 조회

**권한:** Installer, Operator, Admin

**Response (계정 있음):**
```json
{
  "success": true,
  "data": {
    "account": {
      "user_id": "usr_001",
      "name": "홍길동",
      "email": "hong@test.com",
      "phone": "01012345678",
      "role": "OWNER",
      "status": "ACTIVE",
      "app_activation_status": "LIVE",
      "created_at": "2026-03-01T09:00:00Z"
    }
  }
}
```

**Response (계정 없음):**
```json
{
  "success": true,
  "data": {
    "account": null
  }
}
```

### 3.8 고객사 앱 계정 생성

**Endpoint:** `POST /api/admin/owners/{ownerId}/account`

**목적:** 고객사 OWNER 앱 계정 생성 및 초대

**권한:** Operator, Admin

**주의사항:**
- 이미 OWNER 계정이 존재하면 409 오류
- `invite_now = true`면 이메일로 초대 발송 + Cognito 계정 생성
- `invite_now = false`면 DB만 생성 (나중에 초대 가능)

**Request Body:**
```json
{
  "name": "홍길동",
  "email": "hong@test.com",
  "phone": "01012345678",
  "invite_now": true
}
```

**Response (성공 — invite_now: false):**
```json
{
  "success": true,
  "data": {
    "user_id": "usr_001",
    "app_activation_status": "NOT_INVITED"
  }
}
```

**Response (성공 — invite_now: true):**
```json
{
  "success": true,
  "data": {
    "user_id": "usr_001",
    "app_activation_status": "INVITED",
    "temporary_password": "aB3$mK9!xQ2z"
  }
}
```

> `temporary_password`: Cognito 계정 생성 시 자동 발급된 임시 패스워드. 앱 첫 로그인 시 강제 변경됨. 관리자가 이 값을 고객에게 직접 전달해야 함.

**Response (이미 존재):**
```json
{
  "success": false,
  "error": {
    "code": "CONFLICT",
    "message": "Owner account already exists"
  }
}
```

### 3.9 고객사 앱 계정 상태 변경

**Endpoint:** `PATCH /api/admin/owners/{ownerId}/account/status`

**목적:** OWNER 앱 계정 활성화/비활성화 (Cognito 연동)

**권한:** Operator, Admin

**주의사항:**
- `INACTIVE` 처리 시 Cognito 계정도 자동 비활성화 (앱 로그인 불가)
- `ACTIVE` 복구 시 Cognito 계정도 자동 재활성화

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
    "user_id": "usr_001",
    "status": "INACTIVE"
  }
}
```

### 3.10 고객사 앱 계정 삭제

**Endpoint:** `DELETE /api/admin/owners/{ownerId}/account`

**목적:** OWNER 앱 계정 삭제 (soft delete + Cognito 비활성화)

**권한:** Operator, Admin

**Response:**
```json
{
  "success": true,
  "data": null
}
```

---

## 4. Group API

### 4.1 그룹 목록 조회

**Endpoint:** `GET /api/admin/groups`

**목적:** 그룹 목록 조회

**권한:** Installer, Operator, Admin

**Request Query Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| owner_id | string | No | Owner ID |
| keyword | string | No | 그룹명 검색 |
| status | string | No | ACTIVE, INACTIVE, ALL |
| page | number | No | 페이지 번호 |
| size | number | No | 페이지당 건수 |
| sort | string | No | 정렬 |

**Response:**
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "group_id": "grp_001",
        "owner_name": "테스트운수",
        "group_name": "서울지점",
        "member_count": 5,
        "vehicle_count": 20,
        "driver_count": 25,
        "status": "ACTIVE",
        "created_at": "2026-03-01T09:00:00Z"
      }
    ],
    "page_info": { ... }
  }
}
```

### 4.2 그룹 생성

**Endpoint:** `POST /api/admin/groups`

**목적:** 그룹 생성

**권한:** Installer, Operator, Admin

**Request Body:**
```json
{
  "owner_id": "own_001",
  "group_name": "부산지점",
  "status": "ACTIVE"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "group_id": "grp_002"
  }
}
```

### 4.3 그룹 수정

**Endpoint:** `PUT /api/admin/groups/{groupId}`

**목적:** 그룹 정보 수정

**권한:** Operator, Admin

**수정 가능 필드:** `group_name`, `description`, `status`

**수정 불가 필드 (읽기 전용):** `organization_id` — 소속 고객사 변경 불가

**주의사항:**
- `is_default_group` 개념은 현재 백엔드에서 사용하지 않습니다. 고객사 목록의 `default_group_name`은 저장된 플래그가 아니라 가장 먼저 생성된 그룹명을 보여주는 수준입니다.

**Request Body:**
```json
{
  "group_name": "수정그룹명",
  "description": "그룹 설명",
  "status": "ACTIVE"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "group_id": "grp_001"
  }
}
```

### 4.4 그룹 삭제

**Endpoint:** `DELETE /api/admin/groups/{groupId}`

**목적:** 그룹 삭제 (soft delete)

**권한:** Operator, Admin

**Path Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| groupId | string | Yes | 그룹 ID |

**비즈니스 조건:** 소속 차량, 기사, 회원 접근권한 중 하나라도 존재하면 400 오류 반환

**Response (성공):**
```json
{
  "success": true,
  "data": null
}
```

**Response (삭제 불가):**
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Cannot delete group with vehicles, drivers, or member access"
  }
}
```

---

## 5. Device (DTG) API

### 5.1 DTG 목록 조회

**Endpoint:** `GET /api/admin/devices`

**목적:** DTG 기기 목록 조회

**권한:** Installer, Operator, Admin

**Request Query Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| keyword | string | No | 시리얼번호/모델명 검색 |
| owner_id | string | No | Owner ID |
| status | string[] | No | PENDING_ACTIVATION, ACTIVE, REGISTERED, MAPPED, ONLINE, OFFLINE, ERROR |
| installation_status | string | No | NOT_INSTALLED, INSTALLED, VERIFIED, FAILED, REPLACED, ALL |
| page | number | No | 페이지 번호 |
| size | number | No | 페이지당 건수 |
| sort | string | No | 정렬 |

**Response:**
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "device_id": "dev_001",
        "serial_no": "DTG-001",
        "approval_no": "승인12345",
        "product_serial_no": "PROD-001",
        "model_name": "DTG-X1",
        "device_status": "ONLINE",
        "installation_status": "VERIFIED",
        "vehicle_no": "12가3456",
        "last_received_at": "2026-03-25T10:00:00Z",
        "location_summary": "서울 강남구"
      }
    ],
    "page_info": { ... }
  }
}
```

### 5.2 DTG 상세 조회

**Endpoint:** `GET /api/admin/devices/{deviceId}`

**목적:** DTG 상세 정보 및 상태 이력 조회

**권한:** Installer, Operator, Admin

**Response:**
```json
{
  "success": true,
  "data": {
    "device": {
      "device_id": "dev_001",
      "serial_no": "DTG-001",
      "approval_no": "승인12345",
      "product_serial_no": "PROD-001",
      "model_name": "DTG-X1",
      "firmware_version": "1.0.3",
      "line_number": "01012345678",
      "device_status": "ONLINE",
      "installation_status": "VERIFIED",
      "last_received_at": "2026-03-25T10:00:00Z",
      "location_summary": "서울 강남구",
      "installed_at": "2026-03-01T09:00:00Z",
      "install_note": "정상 설치 완료"
    },
    "status_history": [
      {
        "status": "PENDING_ACTIVATION",
        "changed_at": "2026-02-28T10:00:00Z",
        "changed_by": "system"
      },
      {
        "status": "ACTIVE",
        "changed_at": "2026-02-28T10:05:00Z",
        "changed_by": "system"
      }
    ]
  }
}
```

### 5.3 DTG 생성

**Endpoint:** `POST /api/admin/devices`

**목적:** DTG 기기 등록

**권한:** Installer, Operator, Admin

**Request Body:**
```json
{
  "serial_no": "DTG-002",
  "approval_no": "승인12345",
  "product_serial_no": "PROD-002",
  "model_name": "DTG-X1",
  "firmware_version": "1.0.3",
  "line_number": "01098765432",
  "installation_status": "NOT_INSTALLED",
  "install_note": ""
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "device_id": "dev_002"
  }
}
```

### 5.4 DTG 수정

**Endpoint:** `PUT /api/admin/devices/{deviceId}`

**목적:** DTG 정보 수정

**권한:** Operator, Admin

**수정 가능 필드:** `model_name`, `firmware_version`, `line_number`, `installation_status`, `install_note`

**수정 불가 필드 (읽기 전용):** `serial_no`, `approval_no`, `product_serial_no`, `device_status`
- `device_status`(운영 상태)는 시스템이 자동 관리하며 수정 불가
- `serial_no`, `approval_no`, `product_serial_no`는 기기 식별자로 변경 불가

**`installation_status` 허용값:** `INSTALLED`, `UNINSTALLED`, `PENDING`

**Request Body:**
```json
{
  "model_name": "DTG-X2",
  "firmware_version": "1.1.0",
  "line_number": "01011112222",
  "installation_status": "INSTALLED",
  "install_note": "차량에 설치 완료"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "device_id": "dev_001"
  }
}
```

### 5.5 DTG 중복 확인

**Endpoint:** `POST /api/admin/devices/check-duplicate`

**목적:** 시리얼번호/제품일련번호 중복 확인

**Request Body:**
```json
{
  "serial_no": "DTG-002",
  "product_serial_no": "PROD-002"
}
```

### 5.6 DTG 장치 삭제

**Endpoint:** `DELETE /api/admin/devices/{deviceId}`

**목적:** DTG 장치 삭제 (soft delete)

**권한:** Operator, Admin

**Path Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| deviceId | string | Yes | 장치 ID |

**비즈니스 조건:** 차량에 ACTIVE로 연결된 바인딩이 존재하면 400 오류 반환 (바인딩 해제 후 삭제 가능)

**Response (성공):**
```json
{
  "success": true,
  "data": null
}
```

**Response (삭제 불가):**
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Cannot delete device with active vehicle binding"
  }
}
```

---

## 6. Vehicle API

### 6.1 차량 목록 조회

**Endpoint:** `GET /api/admin/vehicles`

**목적:** 차량 목록 조회

**권한:** Installer, Operator, Admin

**Request Query Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| keyword | string | No | 차량번호/차대번호 검색 |
| owner_id | string | No | Owner ID |
| group_id | string | No | 그룹 ID |
| vehicle_status | string[] | No | REGISTERED, DEVICE_LINKED, DRIVER_LINKED, VERIFIED, ACTIVE, INACTIVE |
| running_status | string[] | No | UNKNOWN, STOPPED, IDLING, DRIVING, OFFLINE |
| page | number | No | 페이지 번호 |
| size | number | No | 페이지당 건수 |
| sort | string | No | 정렬 |

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
        "vin": "1HGBH41JXMN109186",
        "owner_name": "테스트운수",
        "group_name": "서울지점",
        "vehicle_status": "VERIFIED",
        "running_status": "DRIVING",
        "installation_status": "VERIFIED",
        "device_serial_no": "DTG-001",
        "primary_driver_name": "홍길동",
        "last_received_at": "2026-03-25T10:00:00Z",
        "location_summary": "서울 강남구"
      }
    ],
    "page_info": { ... }
  }
}
```

### 6.2 차량 상세 조회

**Endpoint:** `GET /api/admin/vehicles/{vehicleId}`

**목적:** 차량 상세 정보 조회

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
      "owner_id": "own_001",
      "owner_name": "테스트운수",
      "group_id": "grp_001",
      "group_name": "서울지점",
      "vehicle_status": "VERIFIED",
      "running_status": "DRIVING",
      "installation_status": "VERIFIED",
      "last_received_at": "2026-03-25T10:00:00Z",
      "location_summary": "서울 강남구",
      "created_at": "2026-03-01T09:00:00Z"
    }
  }
}
```

### 6.3 차량 생성

**Endpoint:** `POST /api/admin/vehicles`

**목적:** 차량 등록

**Request Body:**
```json
{
  "vehicle_no": "34나5678",
  "vin": "2HGBH41JXMN109187",
  "vehicle_type": "BUS",
  "owner_id": "own_001",
  "group_id": "grp_001"
}
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| vehicle_no | string | Yes | 차량번호 ([별표 4] 형식, 최대 20자) |
| vin | string | Yes | 차대번호 (17자리 alphanumeric) |
| vehicle_type | string | No | 자동차유형코드 (TRUCK, BUS, TAXI, SPECIAL, ETC) |
| owner_id | string | Yes | Owner ID |
| group_id | string | Yes | 그룹 ID |

**Response:**
```json
{
  "success": true,
  "data": {
    "vehicle_id": "veh_002"
  }
}
```

 ### 6.4 차량 수정

**Endpoint:** `PUT /api/admin/vehicles/{vehicleId}`

**목적:** 차량 정보 수정

**권한:** Installer, Operator, Admin

**수정 가능 필드:** `vehicle_no`, `vin`, `vehicle_type`, `group_id`

**수정 불가 필드 (읽기 전용):**
- `organization_id` — 소속 고객사 변경 불가
- `vehicle_status`, `running_status`, `install_status` — 시스템이 자동 관리

**주의사항:**
- `vehicle_no`, `vin`은 UNIQUE 제약 — 중복 시 저장 실패 메시지로 안내 (별도 중복 체크 API 없음)
- 값이 없는 필드는 기존 값 유지 (부분 업데이트)

**Request Body:**
```json
{
  "vehicle_no": "34나5678",
  "vin": "2HGBH41JXMN109187",
  "vehicle_type": "BUS",
  "group_id": "grp_002"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "vehicle_id": "veh_001"
  }
}
```

### 6.5 차량 삭제

**Endpoint:** `DELETE /api/admin/vehicles/{vehicleId}`

**목적:** 차량 삭제 (soft delete)

**권한:** Operator, Admin

**Path Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| vehicleId | string | Yes | 차량 ID |

**비즈니스 조건:**
- 차량에 ACTIVE 장치 바인딩이 존재하면 400 오류 반환 (장치 연결 해제 후 삭제 가능)
- 차량에 ACTIVE 기사 배정이 존재하면 400 오류 반환 (기사 배정 해제 후 삭제 가능)

**Response (성공):**
```json
{
  "success": true,
  "data": null
}
```

**Response (삭제 불가 - 장치 연결됨):**
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Cannot delete vehicle with active device binding"
  }
}
```

**Response (삭제 불가 - 기사 배정됨):**
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Cannot delete vehicle with active driver assignment"
  }
}
```

---

## 7. Driver API

### 7.1 기사 목록 조회

**Endpoint:** `GET /api/admin/drivers`

**목적:** 기사 목록 조회

**Request Query Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| keyword | string | No | 기사명/운전자코드 검색 |
| owner_id | string | No | Owner ID |
| group_id | string | No | 그룹 ID |
| account_linked | string | No | YES, NO, ALL |
| status | string | No | ACTIVE, INACTIVE, ALL |
| page | number | No | 페이지 번호 |
| size | number | No | 페이지당 건수 |
| sort | string | No | 정렬 |

**Response:**
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "driver_id": "drv_001",
        "driver_name": "홍길동",
        "phone": "01012345678",
        "driver_code": "DRV20260001",
        "owner_name": "테스트운수",
        "group_name": "서울지점",
        "primary_vehicle_no": "12가3456",
        "account_linked": "YES",
        "app_activation_status": "LIVE",
        "status": "ACTIVE"
      }
    ],
    "page_info": { ... }
  }
}
```

### 7.2 기사 상세 조회

**Endpoint:** `GET /api/admin/drivers/{driverId}`

**Response:**
```json
{
  "success": true,
  "data": {
    "driver": {
      "driver_id": "drv_001",
      "driver_name": "홍길동",
      "phone": "01012345678",
      "driver_code": "DRV20260001",
      "owner_id": "own_001",
      "owner_name": "테스트운수",
      "group_id": "grp_001",
      "group_name": "서울지점",
      "primary_vehicle_no": "12가3456",
      "account_linked": "YES",
      "app_activation_status": "LIVE",
      "status": "ACTIVE",
      "created_at": "2026-03-01T09:00:00Z"
    }
  }
}
```

### 7.3 기사 생성

**Endpoint:** `POST /api/admin/drivers`

**Request Body:**
```json
{
  "driver_name": "김철수",
  "phone": "01098765432",
  "driver_code": "DRV20260002",
  "owner_id": "own_001",
  "group_id": "grp_001"
}
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| driver_name | string | Yes | 기사명 (2~50자) |
| phone | string | Yes | 휴대폰번호 (KR 형식) |
| driver_code | string | No | 운전자코드 (eTAS 전송용, [별표 2] 참조, 최대 20자) |
| owner_id | string | Yes | Owner ID |
| group_id | string | Yes | 그룹 ID |

### 7.4 기사 수정

**Endpoint:** `PUT /api/admin/drivers/{driverId}`

**권한:** Installer, Operator, Admin

**수정 가능 필드:** `driver_name`, `phone`, `driver_code`, `group_id`, `status`

**수정 불가 필드 (읽기 전용):** `organization_id` — 소속 고객사 변경 불가

**주의사항:**
- `driver_code`는 고유값(UNIQUE 인덱스) — 중복 시 저장 오류 발생 (별도 중복 체크 API 없음, 저장 실패 메시지로 안내)
- `group_id` 변경 시 기존 차량 배정은 자동 정리되지 않음 (배정 관계 유지)

**Request Body:**
```json
{
  "driver_name": "김철수",
  "phone": "01011112222",
  "driver_code": "DRV20260003",
  "group_id": "grp_002",
  "status": "ACTIVE"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "driver_id": "drv_001"
  }
}
```

### 7.5 기사 삭제

**Endpoint:** `DELETE /api/admin/drivers/{driverId}`

**목적:** 기사 삭제 (soft delete)

**권한:** Operator, Admin

**Path Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| driverId | string | Yes | 기사 ID |

**비즈니스 조건:** 활성 차량에 배정된 상태이면 400 오류 반환 (배정 해제 후 삭제 가능)

**Response (성공):**
```json
{
  "success": true,
  "data": null
}
```

**Response (삭제 불가):**
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Cannot delete driver with active vehicle assignment"
  }
}
```

---

## 8. Account Linking API

### 8.1 연결 가능한 계정 검색

**Endpoint:** `GET /api/admin/accounts/candidates`

**목적:** 기사-계정 연결 시 기존 계정 후보 검색

**Request Query Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| keyword | string | No | 이름/휴대폰번호 검색 |
| owner_id | string | No | Owner ID |
| group_id | string | No | 그룹 ID |

**Response:**
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "user_id": "usr_001",
        "user_type": "DRIVER",
        "user_name": "홍길동",
        "phone": "01012345678",
        "owner_name": "테스트운수",
        "app_activation_status": "INVITED"
      }
    ],
    "page_info": { ... }
  }
}
```

### 8.2 기존 계정 연결

**Endpoint:** `POST /api/admin/drivers/{driverId}/link-account`

**목적:** 기사와 기존 앱 계정 연결

**Request Body:**
```json
{
  "user_id": "usr_001"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "driver_id": "drv_001",
    "user_id": "usr_001",
    "linked": true
  }
}
```

### 8.3 새 계정 생성 및 연결

**Endpoint:** `POST /api/admin/drivers/{driverId}/create-linked-account`

**Request Body:**
```json
{
  "user_type": "DRIVER",
  "user_name": "신규기사",
  "phone": "01055556666",
  "invite_now": true
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "driver_id": "drv_002",
    "user_id": "usr_005",
    "invite_status": "INVITED"
  }
}
```

---

## 9. Mapping API

### 9.1 매핑 후보 조회

**Endpoint:** `GET /api/admin/mappings/candidates`

**목적:** Quick Mapping 시 선택 가능한 차량/기사/DTG 후보 조회

**Request Query Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| owner_id | string | Yes | Owner ID |
| group_id | string | No | 그룹 ID |
| keyword | string | No | 검색어 |

**Response:**
```json
{
  "success": true,
  "data": {
    "vehicles": [
      {
        "vehicle_id": "veh_001",
        "vehicle_no": "12가3456",
        "vehicle_status": "VERIFIED",
        "is_mapped": false
      }
    ],
    "drivers": [
      {
        "driver_id": "drv_001",
        "driver_name": "홍길동",
        "status": "ACTIVE",
        "is_mapped": true
      }
    ],
    "devices": [
      {
        "device_id": "dev_001",
        "serial_no": "DTG-001",
        "device_status": "ONLINE",
        "is_mapped": true
      }
    ]
  }
}
```

### 9.2 묶음 매핑 저장

**Endpoint:** `POST /api/admin/mappings/bundle`

**목적:** DTG-차량, 차량-기사 묶음 매핑 저장

**Request Body:**
```json
{
  "owner_id": "own_001",
  "group_id": "grp_001",
  "vehicle_id": "veh_001",
  "driver_id": "drv_001",
  "device_id": "dev_001",
  "driver_account_checked": true
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "device_vehicle_mapping_id": "map_dv_001",
    "vehicle_driver_mapping_id": "map_vd_001"
  }
}
```

### 9.3 차량-DTG 매핑 해제

**Endpoint:** `DELETE /api/admin/mappings/device-vehicle/{mappingId}`

**목적:** 활성 상태의 차량-DTG 매핑 해제

**Response:**
```json
{
  "success": true,
  "data": {
    "mapping_id": "map_dv_001",
    "deleted": true
  }
}
```

### 9.4 차량-기사 매핑 해제

**Endpoint:** `DELETE /api/admin/mappings/vehicle-driver/{mappingId}`

**목적:** 활성 상태의 차량-기사 매핑 해제

**Response:**
```json
{
  "success": true,
  "data": {
    "mapping_id": "map_vd_001",
    "deleted": true
  }
}
```

---

## 10. Verification API

### 10.1 검증 목록 조회

**Endpoint:** `GET /api/admin/verifications`

**목적:** 연동 검증 이력 목록 조회

**Request Query Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| keyword | string | No | 차량번호, 장치 S/N, 기사명 검색어 |
| result_status | string[] | No | 현재 예약 필드 (미적용) |
| severity | string[] | No | 현재 예약 필드 (미적용) |
| owner_id | string | No | 현재 예약 필드 (미적용) |
| page | number | No | 페이지 번호 |
| size | number | No | 페이지당 건수 |
| sort | string | No | 현재 `checked_at.desc` 고정 |

**Response:**
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "verification_id": "ver_001",
        "target_type": "VEHICLE",
        "vehicle_no": "12가3456",
        "device_serial_no": "DTG-001",
        "driver_name": "홍길동",
        "target_id": "veh_001",
        "gps_received": true,
        "heartbeat_received": true,
        "device_registered": "PASS",
        "vehicle_mapped": "PASS",
        "driver_mapped": "PASS",
        "gps_check": "PASS",
        "heartbeat_check": "PASS",
        "last_received_at": "2026-03-25T09:58:00Z",
        "result_status": "PASS",
        "severity": "INFO",
        "checked_at": "2026-03-25T10:00:00Z"
      }
    ],
    "page_info": { ... }
  }
}
```

### 10.2 검증 대상 후보 검색

**Endpoint:** `GET /api/admin/verifications/candidates`

**목적:** 차량번호 / 장치 S/N / 기사명으로 검증 가능한 차량 자동완성 검색

**Request Query Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| keyword | string | No | 차량번호, 장치 S/N, 기사명 검색어 |
| size | number | No | 최대 반환 건수 (기본 8) |

**Response:**
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "vehicle_id": "veh_001",
        "vehicle_no": "12가3456",
        "device_serial_no": "DTG-001",
        "driver_name": "홍길동",
        "result_status": "WARNING"
      }
    ]
  }
}
```

### 10.3 검증 상세 조회

**Endpoint:** `GET /api/admin/verifications/{verificationId}`

**Response:**
```json
{
  "success": true,
  "data": {
    "verification": {
      "verification_id": "ver_001",
      "target_type": "VEHICLE",
      "target_id": "veh_001",
      "vehicle_no": "12가3456",
      "result_status": "PASS",
      "severity": "INFO",
      "device_registered": "PASS",
      "vehicle_mapped": "PASS",
      "driver_mapped": "PASS",
      "gps_check": "PASS",
      "heartbeat_check": "PASS",
      "gps_received": true,
      "heartbeat_received": true,
      "last_received_at": "2026-03-25T10:00:00Z",
      "checked_at": "2026-03-25T10:00:00Z"
    }
  }
}
```

### 10.4 검증 실행

**Endpoint:** `POST /api/admin/verifications/run`

**목적:** 선택 차량 검증 또는 미완료 차량 재검증

**Request Body:**
```json
{
  "target_type": "VEHICLE",
  "target_ids": ["veh_001", "veh_002"]
}
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| target_type | string | No | 현재 `VEHICLE`만 사용 |
| target_ids | string[] | No | 선택한 차량 ID 목록. 값이 없으면 ACTIVE 바인딩 중 `PASS`가 아닌 차량만 검증 |

**주의사항:**
- `target_ids`가 존재할 때 빈 배열이면 `400 VALIDATION_ERROR`가 반환됩니다.
- 검증 대상 후보와 기본 재검증 대상은 모두 `vehicle_device_binding.status = 'ACTIVE'`인 차량만 포함합니다.

**Response:**
```json
{
  "success": true,
  "data": {
    "checked_count": 2,
    "checked_at": "2026-03-30T11:00:00Z"
  }
}
```

---

## 11. App Activation API

> **상태:** deprecated 예정. 현재 backend API와 route는 유지되지만 installer-admin-web 사이드바 메뉴에서는 숨김 처리되어 있습니다. 기능은 고객사 상세 / 회원 관리 / 기사 계정 연결 화면으로 점진 이관 예정입니다.

### 11.1 앱 활성화 목록 조회

**Endpoint:** `GET /api/admin/app-activations`

**목적:** 앱 활성화 현황 목록 조회

**Request Query Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| keyword | string | No | 검색어 |
| app_activation_status | string[] | No | NOT_READY, INVITED, FIRST_LOGIN_DONE, LIVE, LOCKED |
| user_type | string[] | No | OWNER, MEMBER, DRIVER |
| owner_id | string | No | Owner ID |
| page | number | No | 페이지 번호 |
| size | number | No | 페이지당 건수 |
| sort | string | No | 정렬 |

**Response:**
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "activation_id": "act_001",
        "user_name": "홍길동",
        "user_type": "DRIVER",
        "phone": "01012345678",
        "owner_name": "테스트운수",
        "vehicle_no": "12가3456",
        "app_activation_status": "LIVE",
        "invited_at": "2026-03-01T09:00:00Z",
        "first_login_at": "2026-03-02T10:00:00Z",
        "live_at": "2026-03-02T10:30:00Z"
      }
    ],
    "page_info": { ... }
  }
}
```

### 11.2 앱 활성화 상세 조회

**Endpoint:** `GET /api/admin/app-activations/{activationId}`

**Response:**
```json
{
  "success": true,
  "data": {
    "activation": {
      "activation_id": "act_001",
      "user_id": "usr_001",
      "user_name": "홍길동",
      "user_type": "DRIVER",
      "phone": "01012345678",
      "owner_name": "테스트운수",
      "app_activation_status": "LIVE",
      "linked_vehicle_no": "12가3456",
      "linked_device_serial_no": "DTG-001"
    },
    "invite_history": [
      {
        "invite_method": "SMS_LINK",
        "invited_at": "2026-03-01T09:00:00Z",
        "invited_by": "installer_001"
      }
    ]
  }
}
```

### 11.3 앱 초대 발송

**Endpoint:** `POST /api/admin/app-activations/{activationId}/invite`

**Request Body:**
```json
{
  "method": "SMS_LINK"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "invite_status": "INVITED",
    "invited_at": "2026-03-25T10:30:00Z"
  }
}
```

### 11.4 앱 초대 재발송

**Endpoint:** `POST /api/admin/app-activations/{activationId}/resend-invite`

**Response:**
```json
{
  "success": true,
  "data": {
    "invite_status": "INVITED",
    "invited_at": "2026-03-25T11:00:00Z"
  }
}
```

### 11.5 서비스 활성화 완료 처리

**Endpoint:** `POST /api/admin/app-activations/{activationId}/mark-live`

**목적:** 앱 첫 로그인 후 서비스 활성화 완료 처리

**Response:**
```json
{
  "success": true,
  "data": {
    "app_activation_status": "LIVE"
  }
}
```

---

## 12. Members API

### 12.1 개념 정의

`member`는 owner에 소속된 하위 운영 계정이다.

- 회사 owner가 담당자/중간 관리자를 추가해 소속 차량/기사/DTG를 관리하거나 모니터링할 수 있도록 하는 계정
- `installer-admin-web` 로그인 대상 아님
- `mobile app` 로그인 대상
- 내부 운영 계정인 `system user`와 분리
- 실제 운전자 계정인 `driver`와 분리

### 12.2 회원 목록 조회

**Endpoint:** `GET /api/admin/members`

**권한:** Installer, Operator, Admin

**Query Params:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| keyword | string | No | 이름, 휴대폰번호 검색 |
| owner_id | string | No | owner 기준 필터 |
| group_id | string | No | group 기준 필터 |
| role | string | No | 역할 필터 |
| app_activation_status | string | No | 앱 활성화 상태 필터 |
| status | string | No | 계정 상태 필터 |
| page | number | No | 페이지 번호 |
| size | number | No | 페이지 크기 |
| sort | string | No | 정렬 |

**Response:**
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "member_id": "mem_001",
        "member_name": "이운영",
        "phone": "01011112222",
        "email": "lee@testlogistics.com",
        "owner_id": "7797a28f-c78c-43c8-8bbe-4d69cbdf8b2a",
        "owner_name": "테스트운수(주)",
        "group_id": "grp_001",
        "group_name": "서울 본부",
        "role": "STAFF",
        "app_activation_status": "INVITED",
        "status": "ACTIVE",
        "created_at": "2026-03-01T09:00:00Z"
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

### 12.3 회원 상세 조회

**Endpoint:** `GET /api/admin/members/{memberId}`

**Response:**
```json
{
  "success": true,
  "data": {
    "member": {
      "member_id": "mem_001",
      "member_name": "이운영",
      "phone": "01011112222",
      "email": "lee@testlogistics.com",
      "owner_id": "own_001",
      "owner_name": "테스트운수(주)",
      "group_id": "grp_001",
      "group_name": "서울 본부",
      "role": "STAFF",
      "app_activation_status": "INVITED",
      "status": "ACTIVE",
      "created_at": "2026-03-01T09:00:00Z"
    }
  }
}
```

### 12.4 회원 등록

**Endpoint:** `POST /api/admin/members`

**Request:**
```json
{
  "member_name": "이운영",
  "phone": "01011112222",
  "email": "lee@testlogistics.com",
  "owner_id": "own_001",
  "group_id": "grp_001",
  "role": "STAFF",
  "invite_now": true
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "member_id": "mem_001",
    "user_id": "mem_001",
    "app_activation_status": "INVITED"
  }
}
```

### 12.5 회원 수정

**Endpoint:** `PUT /api/admin/members/{memberId}`

**권한:** Operator, Admin

**수정 가능 필드:** `member_name`, `phone`, `email`, `group_id`, `role`

**수정 불가 필드 (읽기 전용):** `organization_id`, `user_type`

**주의사항:**
- `status` 변경은 이 API가 아닌 **`PATCH /api/admin/members/{memberId}/status`** 사용 (Cognito 연동 포함)
- `group_id` 변경 시 기존 그룹 접근권한이 **전부 삭제**되고 새 그룹 1개로 재설정됨 (다중 그룹 선택 불가)
- `role`은 `STAFF` 고정 운영 권장 (`OWNER` 계정과 혼용 금지)

**Request:**
```json
{
  "member_name": "이운영",
  "phone": "01011112222",
  "email": "lee@testlogistics.com",
  "group_id": "grp_002",
  "role": "STAFF"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "member_id": "mem_001"
  }
}
```

### 12.6 회원 초대 발송

**Endpoint:** `POST /api/admin/members/{memberId}/invite`

**Request:**
```json
{
  "method": "EMAIL"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "member_id": "mem_001",
    "app_activation_status": "INVITED",
    "invited_at": "2026-03-28T10:00:00Z"
  }
}
```

### 12.7 회원 초대 재발송

**Endpoint:** `POST /api/admin/members/{memberId}/resend-invite`

**Response:**
```json
{
  "success": true,
  "data": {
    "member_id": "mem_001",
    "app_activation_status": "INVITED",
    "invited_at": "2026-03-28T10:30:00Z"
  }
}
```

### 12.8 회원 상태 변경

**Endpoint:** `PATCH /api/admin/members/{memberId}/status`

**권한:** Operator, Admin

**주의사항:**
- 회원 `status` 변경은 반드시 이 API를 사용 (`PUT /members/{memberId}`에 status 포함 불가)
- `INACTIVE` 처리 시 Cognito 계정도 자동 비활성화됨 (앱 로그인 불가)
- `ACTIVE` 복구 시 Cognito 계정도 자동 재활성화됨

**허용값:** `ACTIVE`, `INACTIVE`

**Request:**
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
    "member_id": "mem_001",
    "status": "INACTIVE"
  }
}
```

### 12.9 회원 삭제

**Endpoint:** `DELETE /api/admin/members/{memberId}`

**목적:** 회원 삭제 (soft delete + Cognito 비활성화)

**권한:** Operator, Admin

**Path Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| memberId | string | Yes | 회원 ID |

**비즈니스 조건:** 차단 조건 없음. 삭제 시 그룹 접근권한 자동 제거 및 Cognito 계정 비활성화 처리

**Response (성공):**
```json
{
  "success": true,
  "data": null
}
```

### 12.10 업무 규칙

- member는 반드시 하나의 owner에 속해야 한다.
- member는 최소 1개 group에 속해야 한다.
- 개인 owner는 member 없이 운영 가능하다.
- 회사 owner는 여러 member를 둘 수 있다.
- member는 자신에게 허용된 owner/group 범위 안에서만 차량, 기사, DTG를 조회할 수 있다.

### 12.11 현재 1차 구현 메모

- 1차 구현은 `app_user` + `user_group_access` 기준으로 member 계정을 관리한다.
- 저장 시 `user_type = STAFF`, `role = STAFF`를 사용한다.
- `app_activation_status`는 전용 저장 컬럼이 아니라 현재 기준으로 다음과 같이 계산한다.
  - `last_login_at IS NOT NULL` → `LIVE`
  - 그 외 → `NOT_INVITED`
- member 세부 역할 체계는 후속 스키마 확장 전까지 `STAFF` 단일값으로 운영한다.

---

## 13. System API

### 13.1 시스템 사용자 목록 조회

**Endpoint:** `GET /api/admin/system/users`

**목적:** Installer Admin 내부 사용자 목록 조회

**현재 구현 메모:** 시스템 사용자의 source of truth는 실제 DB가 아니라 Cognito User Pool이다. 목록은 Cognito 사용자와 Cognito group 기준으로 구성된다.

**권한:** Admin

**Request Query Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| keyword | string | No | 이름/이메일 검색 |
| role | string[] | No | Installer, Operator, Admin |
| status | string | No | ACTIVE, INACTIVE, LOCKED, ALL |
| page | number | No | 페이지 번호 |
| size | number | No | 페이지당 건수 |
| sort | string | No | 정렬 |

**Response:**
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "user_id": "sys_001",
        "user_name": "관리자",
        "email": "admin@trackcar.co.kr",
        "role": "Admin",
        "status": "ACTIVE",
        "last_login_at": "2026-03-25T09:00:00Z",
        "created_at": "2026-01-01T00:00:00Z"
      }
    ],
    "page_info": { ... }
  }
}
```

### 13.2 시스템 사용자 생성

**Endpoint:** `POST /api/admin/system/users`

**권한:** Admin

**Request Body:**
```json
{
  "user_name": "신규관리자",
  "email": "newadmin@trackcar.co.kr",
  "role": "Operator",
  "status": "ACTIVE"
}
```

**동작 방식:**
- Cognito `AdminCreateUser` 기반으로 계정을 생성한다.
- 무작위 임시 비밀번호를 발급한다.
- 첫 로그인 시 `NEW_PASSWORD_REQUIRED` 챌린지를 통해 비밀번호 변경이 강제된다.

**Response:**
```json
{
  "success": true,
  "data": {
    "user_id": "6408bd9c-3051-70e1-7054-0477eb79bd1e",
    "login_username": "newadmin@trackcar.co.kr",
    "temporary_password": "Cy$5dbegHLYX",
    "password_change_required": true,
    "role": "Operator",
    "status": "ACTIVE"
  }
}
```

### 13.3 시스템 사용자 최초 로그인

- 임시 비밀번호로 로그인하면 Cognito가 `NEW_PASSWORD_REQUIRED`를 반환한다.
- 프론트는 새 비밀번호 입력 UI를 표시하고 `completeNewPasswordChallenge`를 수행한다.
- 비밀번호 변경이 완료되면 정상 로그인 세션으로 전환된다.

---

## 14. 공통 타입 정의

### 14.1 Status Enum Values

|Enum|값|
|---|---|
| **owner_status** | ACTIVE, INACTIVE, LOCKED |
| **owner_type** | PERSONAL, BUSINESS |
| **vehicle_status** | REGISTERED, DEVICE_LINKED, DRIVER_LINKED, VERIFIED, ACTIVE, INACTIVE |
| **running_status** | UNKNOWN, STOPPED, IDLING, DRIVING, OFFLINE |
| **installation_status** | NOT_INSTALLED, INSTALLED, VERIFIED, FAILED, REPLACED |
| **device_status** | PENDING_ACTIVATION, ACTIVE, REGISTERED, MAPPED, ONLINE, OFFLINE, ERROR |
| **verification_status** | NOT_CHECKED, CHECKING, PASS, WARNING, FAIL |
| **severity** | INFO, WARNING, CRITICAL |
| **app_activation_status** | NOT_READY, INVITED, FIRST_LOGIN_DONE, LIVE, LOCKED |
| **user_type** | OWNER, MEMBER, DRIVER |
| **admin_role** | Installer, Operator, Admin |
| **group_status** | ACTIVE, INACTIVE |
| **invite_method** | SMS_LINK, TEMP_PASSWORD, LATER |
| **driver_account_status** | YES, NO |

### 14.2 Page Info

```json
{
  "page": 1,
  "size": 20,
  "total": 100,
  "has_next": true
}
```

---

## 15. 변경 이력 (Changelog)

- **v1.5 (2026-03-29):**
  - 실제 Admin API Gateway base URL을 `a1xpskzj48 ... /v1/api/admin` 기준으로 수정
  - Vite 환경변수명(`VITE_*`) 기준으로 문서화
  - system user 최초 로그인 `NEW_PASSWORD_REQUIRED` 흐름 추가
  - owner 예시의 `owner_id`를 실제 UUID 형식으로 보정

- **v1.4 (2026-03-29):**
  - System Users를 Cognito 기반 source of truth로 명시
  - 시스템 사용자 생성 응답에 `login_username`, `temporary_password`, `password_change_required` 추가
  - 최초 로그인 `NEW_PASSWORD_REQUIRED` 흐름 문서화
  - Members 1차 구현 메모에서 `cognito_user_id` 의존 설명 제거

- **v1.3 (2026-03-28):**
  - Members API를 정식 계약으로 추가
  - `members`를 owner 하위 운영 계정 도메인으로 정의
  - `members`와 `system users`, `drivers`, `app activations`의 역할 분리 반영

- **v1.2 (2026-03-28):**
  - `/members` 화면은 현재 프론트 프로토타입이며 정식 백엔드 계약 범위가 아님을 명시
  - 현재 구현 기준의 API 지원 범위 주의사항 추가

- **v1.1 (2026-03-28):**
  - Mapping API에 매핑 해제 엔드포인트 2종 추가
  - 백엔드 AdminHandler 반영 범위에 맞춰 스펙 보강

- **v1.0 (2026-03-25):**
  - 최초 API Spec 문서 작성
  - Installer Admin Web 화면 상세 설계서 기반 API 정의
  - Dashboard, Owner, Group, Device, Vehicle, Driver, Mapping, Verification, App Activation, System API 포함

---

## 16. 참고 문서

| 문서명 | 설명 |
|--------|------|
| TrackCar Installer Admin 요구사항 및 기능 명세서 | 기능/요구사항 정의 |
| TrackCar Installer Admin 화면 상세 설계서 | 화면별 상세 설계 |
| TrackCar Installer Admin 프로세스 및 IA 설계서 | 프로세스/메뉴 구조 |
| TrackCar 플랫폼 아키텍처 상세 설계 | Backend 아키텍처 |
| TrackCar Ingest 검증 명세서 | Ingest 파이프라인 |
| TrackCar 외부 연계 전송 처리 명세서 | eTAS 전송 |
