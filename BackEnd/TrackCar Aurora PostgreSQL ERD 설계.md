# TrackCar Aurora PostgreSQL ERD 설계

- 작성일: 2026-03-31
- 상태: Draft
- 기준 문서: `TrackCar Aurora PostgreSQL 물리 스키마 명세서.md`

---

## 1. 문서 목적

본 문서는 TrackCar 백엔드의 Aurora PostgreSQL 핵심 엔터티 관계를 **Mermaid ERD** 형식으로 정리한 문서입니다.

용도:
- 설치 운영 기능의 도메인 관계를 빠르게 파악
- Admin/Mobile API 설계 시 테이블 관계 참고
- 매핑/검증/운행/전송 파이프라인의 데이터 흐름 이해

---

## 2. 범위와 주의사항

- 본 ERD는 **핵심 논리 관계 이해용 설계 문서**입니다. 전체 컬럼 상세와 물리 제약은 `TrackCar Aurora PostgreSQL 물리 스키마 명세서.md`를 기준으로 봅니다.
- 실제 구현에서는 일부 명칭 드리프트가 있습니다.
  - 스키마 문서: `app_user_group_scope`
  - 일부 코드 흔적: `user_group_access`
- 시스템 사용자(`system/users`)는 Aurora가 아니라 **Cognito가 source of truth** 이므로 이 ERD의 `app_user`와는 별도 관점으로 봐야 합니다.
- `verification`은 현재 운영 기능상 사용되는 **논리 테이블 개념**으로 포함했습니다. 물리 스키마 문서와 완전히 동기화되지 않았을 수 있으므로 FK처럼 엄격하게 해석하지 마세요.
- 본 문서는 다음 영역을 **범위 밖**으로 둡니다: `subscription`, `dtg_interface_catalog`, `outbound_profile`, `outbound_interface_catalog`

---

## 3. 핵심 운영 도메인 ERD

```mermaid
erDiagram
    organization ||--o{ team : has
    organization ||--o{ vehicle : owns
    organization ||--o{ driver : employs
    organization ||--o{ app_user : has

    team o|--o{ vehicle : groups_optional
    team o|--o{ driver : groups_optional
    team ||--o{ app_user_group_scope : grants

    app_user ||--o{ app_user_group_scope : scoped_to

    vehicle ||--o{ vehicle_device_binding : bound_to
    device  ||--o{ vehicle_device_binding : assigned_to

    vehicle ||--o{ vehicle_driver_link : linked_to
    driver  ||--o{ vehicle_driver_link : assigned_to

    vehicle o|--o{ verification : verified_by_logically

    organization {
        varchar organization_id PK
        varchar name
        varchar owner_type
        varchar business_no
        varchar transport_biz_no
        varchar phone
        varchar email
        varchar status
        timestamptz deleted_at
    }

    team {
        varchar group_id PK
        varchar organization_id FK
        varchar name
        varchar description
        varchar status
        timestamptz deleted_at
    }

    vehicle {
        varchar vehicle_id PK
        varchar organization_id FK
        varchar group_id FK
        varchar vehicle_no
        varchar vin
        varchar vehicle_type
        varchar vehicle_status
        varchar running_status
        varchar install_status
        timestamptz deleted_at
    }

    device {
        varchar device_id PK
        varchar serial_no
        varchar approval_no
        varchar product_serial_no
        varchar device_status
        varchar install_status
        timestamptz last_received_at
        timestamptz deleted_at
    }

    driver {
        varchar driver_id PK
        varchar organization_id FK
        varchar group_id FK
        varchar name
        varchar phone
        varchar driver_code
        varchar status
        timestamptz deleted_at
    }

    app_user {
        varchar user_id PK
        varchar organization_id FK
        varchar cognito_user_id
        varchar name
        varchar email
        varchar role
        varchar scope_type
        varchar status
        timestamptz deleted_at
    }

    app_user_group_scope {
        varchar scope_id PK
        varchar user_id FK
        varchar group_id FK
    }

    vehicle_device_binding {
        varchar binding_id PK
        varchar vehicle_id FK
        varchar device_id FK
        varchar status
        timestamptz bound_at
        timestamptz unbound_at
    }

    vehicle_driver_link {
        varchar link_id PK
        varchar vehicle_id FK
        varchar driver_id FK
        varchar status
        timestamptz linked_at
        timestamptz unlinked_at
    }

    verification {
        varchar verification_id PK
        varchar target_type
        varchar target_id
        varchar result_status
        varchar severity
        timestamptz last_received_at
        timestamptz checked_at
    }
```

### 해설

- `organization`이 고객사 루트 엔터티입니다.
- `team`은 고객사 하위 그룹입니다.
- `vehicle`, `driver`, `app_user`는 모두 고객사 소속입니다.
- 실제 설치 운영의 핵심은 `vehicle_device_binding` + `vehicle_driver_link` 두 매핑 테이블입니다.
- 연동 검증은 현재 `verification.target_id = vehicle.vehicle_id` 기준으로 차량 단위 상태를 저장합니다. 다만 `target_id`는 다형적 필드라 물리 FK로 강제된다는 의미는 아닙니다.

---

## 4. 운행 / 위치 / 알림 도메인 ERD

```mermaid
erDiagram
    vehicle ||--o{ trip_meta : has
    driver  o|--o{ trip_meta : drives_optional

    trip_meta ||--|| trip_route_summary : summarized_as
    trip_meta ||--o{ trip_event : emits
    trip_meta ||--o{ alert_event : triggers

    vehicle ||--o{ trip_route_summary : has
    vehicle ||--o{ trip_event : emits
    vehicle o|--o{ alert_event : raises_optional

    trip_meta {
        varchar trip_id PK
        varchar vehicle_id FK
        varchar driver_id
        timestamptz started_at
        timestamptz ended_at
        decimal distance_km
        integer duration_min
        date trip_date
        varchar status
    }

    trip_route_summary {
        varchar route_id PK
        varchar trip_id FK
        varchar vehicle_id FK
        jsonb route_points
        varchar location_summary
    }

    trip_event {
        varchar event_id PK
        varchar trip_id FK
        varchar vehicle_id FK
        varchar event_type
        timestamptz occurred_at
        integer duration_min
        decimal latitude
        decimal longitude
    }

    alert_event {
        varchar alert_id PK
        varchar vehicle_id FK
        varchar trip_id FK
        varchar alert_type
        varchar severity
        varchar status
        timestamptz occurred_at
        timestamptz resolved_at
    }
```

### 해설

- `trip_meta`가 운행 세션의 루트입니다.
- `trip_route_summary`는 전체 raw point 저장소가 아니라 **모바일/조회용 요약 경로**입니다.
- `trip_event`는 시동/정차/과속 등 주요 이벤트를 저장합니다.
- `alert_event`는 차량/운행 기반 이상 상황을 저장합니다. `vehicle_id`가 nullable이라 차량 비귀속 이벤트 가능성도 열어둡니다.

---

## 5. 외부 전송 / eTAS 도메인 ERD

```mermaid
erDiagram
    vehicle ||--o{ etas_transmission_target : scheduled_for
    vehicle ||--o{ etas_transmission_log : transmitted_as
    vehicle o|--o{ outbound_transmission : sent_to_external_optional
    organization ||--o{ outbound_transmission : tenant_scope_logically
    etas_transmission_target o|--o{ etas_transmission_log : results_in_logically

    etas_transmission_target {
        varchar target_id PK
        varchar vehicle_id FK
        date business_date
        varchar vehicle_no
        varchar vin
        varchar transport_biz_no
        varchar driver_code
        varchar status
        integer attempt_count
    }

    etas_transmission_log {
        varchar transmission_id PK
        varchar vehicle_id FK
        date business_date
        varchar generated_file_name
        varchar status
        varchar idempotency_key
        timestamptz transmitted_at
    }

    outbound_transmission {
        varchar transmission_id PK
        varchar interface_code
        varchar trigger_policy
        varchar tenant_id
        varchar vehicle_id FK
        varchar resource_key
        varchar status
        integer attempt_count
        timestamptz transmitted_at
    }
```

### 해설

- `etas_transmission_target`은 일자별 전송 대상 생성 테이블입니다.
- `etas_transmission_log`는 실제 eTAS 전송 이력과 오류를 남깁니다.
- `outbound_transmission`은 eTAS 외 외부 인터페이스 전송 이력을 위한 공통 로그 성격입니다.
- `tenant_id`와 eTAS target→log 관계는 현재 운영 해석상 중요한 논리 연결이며, 물리 FK 제약과 1:1로 동일하다는 의미는 아닙니다.

---

## 6. 설치 운영 관점 핵심 흐름

```mermaid
flowchart LR
    A[organization 생성] --> B[team 생성]
    A --> C[vehicle 생성]
    A --> D[driver 생성]
    A --> E[app_user 생성]

    C --> F[vehicle_device_binding]
    D --> G[vehicle_driver_link]
    F --> H[IoT telemetry ingest]
    G --> H
    H --> I[verification]
    H --> J[trip_meta / trip_event / alert_event]
    J --> K[eTAS target/log]
```

### 해설

- 설치 운영의 최소 완료 단위는 **고객사/차량/장치/기사 생성 + 매핑 완료**입니다.
- 이후 실제 데이터 수신이 되면 ingest 파이프라인이 운행/알림 데이터를 만들고,
- 설치 담당자는 `verification`을 통해 통신 검증 상태를 확인합니다.

---

## 7. 참고

- 상세 컬럼/인덱스/enum 정의: `TrackCar Aurora PostgreSQL 물리 스키마 명세서.md`
- Admin API 응답/엔드포인트: `TrackCar AdminHandler 구현 설계.md`
- IoT 및 ingest 흐름: `DTG 단말기 ↔ TrackCar IoT Core 연동 요구사항.md`, `TrackCar Ingest 검증 명세서.md`
