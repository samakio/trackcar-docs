# TrackCar Mobile App 화면 상세 설계서

- 작성일: 2026-03-25
- 버전: v1.0
- 상태: 초안

---

## 1. 문서 개요

### 1.1 목적

본 문서는 AI Agent가 바로 구현에 활용할 수 있도록 TrackCar Mobile App의 화면을 기능, 상태, 데이터, 액션, 권한, 검증 중심으로 정의한다.

### 1.2 전제

- 서비스: TrackCar Mobile App
- 도메인: 차량관제 / 운행관리 / 알림 / 조직운영
- 역할:
  - OWNER: 조직 대표, 담당자/그룹/차량-기사 연결 운영 가능
  - STAFF: 조회/알림 중심 사용자
- 모바일 앱은 고객용 Self-Service 채널이다.
- 차량-기사 관계는 일자별 배차가 아니라 상시 연결 기준으로 관리한다.
- 실시간 화면은 30초 polling + 수동 새로고침을 지원하고 마지막 갱신 시각을 표시한다.
- 내부 설치/개통/검증 기능은 내부 Admin Web 범위이며 본 문서에서 제외한다.

### 1.3 공통 상태 코드

- 차량 상태: REGISTERED, DEVICE_LINKED, DRIVER_LINKED, VERIFIED, ACTIVE, INACTIVE
- 운행 상태: RUNNING, IDLE, OFFLINE, WARNING
- 설치 상태: NOT_INSTALLED, INSTALLED, VERIFIED, FAILED, REPLACED
- 알림 심각도: CRITICAL, HIGH, MEDIUM, LOW
- DTG 상태: PENDING_ACTIVATION, ACTIVE, REGISTERED, MAPPED, ONLINE, OFFLINE, ERROR

### 1.4 공통 UI 규칙

- 목록 화면은 무한 스크롤(infinite scroll) 기반 페이지네이션을 지원한다.
- 실시간 상태가 포함된 화면은 마지막 갱신 시각과 수동 새로고침 버튼을 표시한다.
- Owner와 Staff의 기능 차이는 UI에서 명확히 구분한다.
- 읽기 전용 필드는 `readonly: true`로 명시한다.
- Owner 전용 기능은 `visible: OWNER only`로 표기한다.

전제:
- 서비스: TrackCar Mobile
- 도메인: 차량관제 / 운행관리 / 알림 / 조직운영
- 역할:
  - OWNER: 조직 대표, 담당자/그룹/차량-기사 연결 운영 가능
  - STAFF: 조회/알림 중심 사용자
- 모바일 앱은 고객용 Self-Service 채널이다.
- 차량-기사 관계는 일자별 배차가 아니라 상시 연결 기준으로 관리한다.
- 실시간 화면은 30초 polling + 수동 새로고침을 지원하고 마지막 갱신 시각을 표시한다.
- 내부 설치/개통/검증 기능은 내부 Admin Web 범위이며 본 문서에서 제외한다.

### 홈 대시보드
```yaml
screen:
  id: SCR-HOME-001
  name: 홈 대시보드
  service: TrackCar Mobile
  domain: 관제
  route: /home
  menu_path: Home
  roles: [OWNER, STAFF]
  purpose: 권한 범위 내 차량 운영 현황, 알림 현황, 담당자 연결 상태를 실시간으로 요약 조회한다.

layout:
  type: dashboard
  platform: mobile
  navigation:
    header: 상단 타이틀 + 조직 선택 + 마지막 갱신 시각 + 새로고침
    body: KPI 카드 + 조치 필요 차량 + 최근 알림 + 빠른 이동
    footer: 하단 탭바

data_scope:
  primary_entity: organization_dashboard
  params: [organization_id, group_id]
  filters: [group_id, vehicle_status, trip_status]
  sort: [priority desc, last_received_at asc]
  pagination:
    enabled: false

components:
  - id: cmp_org_selector
    type: select
    label: 조직 선택
    required: true
    readonly: false
    visible: true
    enabled: true
    options: OWNER는 소속 조직 목록, STAFF는 접근 가능한 조직 목록
    data_binding: session.organization_id
    description: 조직 전환 시 KPI와 목록 데이터를 재조회한다.
  - id: cmp_last_refresh
    type: text
    label: 마지막 갱신 시각
    required: false
    readonly: true
    visible: true
    enabled: true
    options: []
    data_binding: dashboard.last_refresh_at
    description: 마지막 화면 갱신 시각 표시
  - id: cmp_refresh_button
    type: button
    label: 지금 새로고침
    required: false
    readonly: false
    visible: true
    enabled: true
    options: [manual_refresh]
    data_binding: none
    description: 수동 재조회
  - id: cmp_group_filter
    type: chip_filter
    label: 그룹 필터
    required: false
    readonly: false
    visible: true
    enabled: true
    options: [전체, 그룹별]
    data_binding: query.group_id
    description: 그룹 기준 필터
  - id: cmp_kpi_assigned
    type: kpi_card
    label: 담당자 연결 차량
    required: false
    readonly: true
    visible: true
    enabled: true
    options: []
    data_binding: dashboard.assigned_vehicle_count
    description: active 차량-기사 연결 차량 수
  - id: cmp_kpi_unassigned
    type: kpi_card
    label: 담당자 미연결 차량
    required: false
    readonly: true
    visible: true
    enabled: true
    options: []
    data_binding: dashboard.unassigned_vehicle_count
    description: 연결 없는 예외 차량 수
  - id: cmp_kpi_running
    type: kpi_card
    label: 운행중 차량
    required: false
    readonly: true
    visible: true
    enabled: true
    options: []
    data_binding: dashboard.running_vehicle_count
    description: trip_status RUNNING 차량 수
  - id: cmp_kpi_offline
    type: kpi_card
    label: 오프라인 차량
    required: false
    readonly: true
    visible: true
    enabled: true
    options: []
    data_binding: dashboard.offline_vehicle_count
    description: vehicle_status OFFLINE 차량 수
  - id: cmp_kpi_alert
    type: kpi_card
    label: 심각/높음 알림
    required: false
    readonly: true
    visible: true
    enabled: true
    options: []
    data_binding: dashboard.high_alert_count
    description: severity가 CRITICAL 또는 HIGH인 미해결 알림 수
  - id: cmp_risk_vehicle_list
    type: list
    label: 조치 필요 차량
    required: false
    readonly: true
    visible: true
    enabled: true
    options: [상태뱃지, 기사명, 그룹명, 최종수신시각]
    data_binding: dashboard.risk_vehicles
    description: 오프라인, 미연결, 심각 알림 차량 우선 노출
  - id: cmp_recent_alerts
    type: list
    label: 최근 알림
    required: false
    readonly: true
    visible: true
    enabled: true
    options: [severity, 차량, 발생시각, 상태]
    data_binding: dashboard.recent_alerts
    description: 최근 알림 5건
  - id: cmp_quick_links
    type: action_grid
    label: 빠른 이동
    required: false
    readonly: false
    visible: true
    enabled: true
    options: [차량목록, 알림센터, 워크스페이스]
    data_binding: none
    description: 주요 화면 바로가기

actions:
  - trigger: 화면 진입
    target: 홈 대시보드
    behavior: 조직/권한 기준 KPI와 목록 조회
    api_ref: GET /v1/mobile/dashboard
    success_result: 대시보드 렌더링 및 마지막 갱신 시각 표시
    error_result: 오류 배너와 재시도 버튼 표시
  - trigger: 새로고침 버튼 탭
    target: 홈 대시보드
    behavior: 즉시 재조회 후 갱신 시각 업데이트
    api_ref: GET /v1/mobile/dashboard
    success_result: 최신 KPI 및 목록 반영
    error_result: 토스트 '데이터를 불러오지 못했습니다.'
  - trigger: KPI 카드 탭
    target: 차량 목록
    behavior: 상태 필터를 적용한 차량목록 화면으로 이동
    api_ref: none
    success_result: 필터된 목록 진입
    error_result: none

states:
  initial: 스켈레톤 카드와 리스트 표시
  normal: KPI, 조치 필요 차량, 최근 알림 정상 표시
  loading: 상단 로딩 인디케이터와 pull-to-refresh 표시
  empty: 표시할 운영 데이터가 없습니다.
  error: 대시보드 정보를 불러오지 못했습니다.
  no_permission: 접근 가능한 조직 정보가 없습니다.
  conditional:
    - OWNER만 워크스페이스 링크 표시
    - STAFF는 읽기 전용
    - 오프라인 차량 존재 시 상단 경고 배너 표시

validation:
  organization_id:
    rule: 필수
    message: 조직을 선택해 주세요.
  group_id:
    rule: 선택 항목
    message: 없음

permissions:
  view: OWNER, STAFF
  create: none
  update: none
  delete: none
  field_control:
    - OWNER: organization 전환 가능
    - STAFF: 허용된 조직/그룹만 조회 가능

api:
  method: GET
  endpoint: /v1/mobile/dashboard
  purpose: 모바일 홈 대시보드 조회
  request:
    query:
      organization_id: string
      group_id: string|null
  response:
    dashboard:
      last_refresh_at: datetime
      assigned_vehicle_count: number
      unassigned_vehicle_count: number
      running_vehicle_count: number
      offline_vehicle_count: number
      high_alert_count: number
      risk_vehicles: array
      recent_alerts: array

navigation_rules:
  entry_points: [앱 로그인 직후 기본 진입, 하단 탭 Home]
  exits: [차량 상세, 알림 센터, 워크스페이스, 차량 목록]

ui_text:
  title: 운영 현황
  empty_message: 표시할 운영 데이터가 없습니다.
  error_message: 대시보드 정보를 불러오지 못했습니다.
  toast: [데이터가 갱신되었습니다., 데이터를 불러오지 못했습니다.]
  confirm_message: []

test_points:
  - 권한별 OWNER/STAFF 메뉴 노출 차이 확인
  - 수동 새로고침 시 마지막 갱신 시각 변경 확인
  - KPI 탭 시 필터 전달 확인
  - 오프라인 차량 발생 시 조치 필요 차량 우선 노출 확인
```

### 차량 목록
```yaml
screen:
  id: SCR-VEH-001
  name: 차량 목록
  service: TrackCar Mobile
  domain: 차량
  route: /vehicles
  menu_path: Vehicles > 차량 목록
  roles: [OWNER, STAFF]
  purpose: 권한 범위 내 차량을 검색, 필터, 정렬하여 조회하고 상세 화면으로 진입한다.

layout:
  type: list
  platform: mobile
  navigation:
    header: 상단 타이틀 + 검색 + 필터 버튼
    body: 상태 요약칩 + 차량 리스트
    footer: 하단 탭바

data_scope:
  primary_entity: vehicle
  params: [organization_id, keyword, group_id, vehicle_status, trip_status, install_status, assigned_status, page, size, sort]
  filters: [group_id, vehicle_status, trip_status, install_status, assigned_status]
  sort: [last_received_at desc, vehicle_no asc, updated_at desc]
  pagination:
    enabled: true
    type: infinite_scroll

components:
  - id: cmp_search
    type: search
    label: 차량번호/별칭 검색
    required: false
    readonly: false
    visible: true
    enabled: true
    options: debounce 500ms
    data_binding: query.keyword
    description: 차량번호, 차량별칭, 기사명 검색 지원
  - id: cmp_filter_sheet_button
    type: button
    label: 필터
    required: false
    readonly: false
    visible: true
    enabled: true
    options: [bottom_sheet]
    data_binding: none
    description: 차량 상태, 운행 상태, 설치 상태, 그룹 필터 열기
  - id: cmp_sort_selector
    type: select
    label: 정렬
    required: false
    readonly: false
    visible: true
    enabled: true
    options: [최종수신 최신순, 차량번호순, 최근수정순]
    data_binding: query.sort
    description: 목록 정렬 기준 선택
  - id: cmp_vehicle_list
    type: list
    label: 차량 리스트
    required: false
    readonly: true
    visible: true
    enabled: true
    options: [상태뱃지, 기사명, 그룹명, 최종수신시각, 현재위치 요약]
    data_binding: vehicles.items
    description: 카드형 리스트 표시
  - id: cmp_bottom_sheet_filter
    type: bottom_sheet
    label: 차량 필터
    required: false
    readonly: false
    visible: true
    enabled: true
    options: [그룹, 차량상태, 운행상태, 설치상태, 담당자연결여부]
    data_binding: query.*
    description: 목록 조건 상세 조정

actions:
  - trigger: 화면 진입
    target: 차량 목록
    behavior: 기본 정렬과 기본 필터로 첫 페이지 조회
    api_ref: GET /v1/mobile/vehicles
    success_result: 목록 렌더링
    error_result: 오류 상태 표시
  - trigger: 검색 입력
    target: 차량 목록
    behavior: 입력 종료 후 500ms 뒤 재조회
    api_ref: GET /v1/mobile/vehicles
    success_result: 검색 결과 반영
    error_result: 토스트 '검색 결과를 불러오지 못했습니다.'
  - trigger: 필터 적용
    target: 차량 목록
    behavior: 선택 조건으로 목록 재조회
    api_ref: GET /v1/mobile/vehicles
    success_result: 필터 적용 결과 표시
    error_result: 토스트 '필터 적용에 실패했습니다.'
  - trigger: 차량 카드 탭
    target: 차량 상세
    behavior: 선택 차량 상세 이동
    api_ref: none
    success_result: 상세 진입
    error_result: none

states:
  initial: 스켈레톤 차량 카드 6건 표시
  normal: 조건별 결과 목록 표시
  loading: 상단 로딩 또는 하단 infinite loader 표시
  empty: 조건에 맞는 차량이 없습니다.
  error: 차량 목록을 불러오지 못했습니다.
  no_permission: 조회 가능한 차량이 없습니다.
  conditional:
    - STAFF는 허용된 그룹/차량만 조회
    - OWNER는 전체 또는 그룹별 조회 가능
    - 오프라인 차량은 카드 상단 붉은 상태 표시

validation:
  keyword:
    rule: 최대 50자
    message: 검색어는 50자 이하로 입력해 주세요.
  page:
    rule: 1 이상
    message: 올바르지 않은 페이지입니다.
  size:
    rule: 10~50
    message: 페이지 크기가 올바르지 않습니다.

permissions:
  view: OWNER, STAFF
  create: none
  update: none
  delete: none
  field_control:
    - OWNER: 전체 조직 범위 필터 가능
    - STAFF: 접근 권한 없는 그룹 필터 숨김

api:
  method: GET
  endpoint: /v1/mobile/vehicles
  purpose: 차량 목록 조회
  request:
    query:
      organization_id: string
      keyword: string|null
      group_id: string|null
      vehicle_status: string|null
      trip_status: string|null
      install_status: string|null
      assigned_status: string|null
      page: number
      size: number
      sort: string
  response:
    items: array
    page: number
    size: number
    total: number
    has_next: boolean

navigation_rules:
  entry_points: [하단 탭 Vehicles, 홈 KPI 카드 탭]
  exits: [차량 상세, 필터 바텀시트, 홈]

ui_text:
  title: 차량 목록
  empty_message: 조건에 맞는 차량이 없습니다.
  error_message: 차량 목록을 불러오지 못했습니다.
  toast: [필터가 적용되었습니다., 검색 결과를 불러오지 못했습니다.]
  confirm_message: [필터를 초기화할까요?]

test_points:
  - 검색어 debounce 동작 확인
  - 복합 필터 적용 시 API 파라미터 정확성 확인
  - STAFF 권한에서 비허용 차량 노출 차단 확인
  - 상태 뱃지와 최종 수신 시각 표기 확인
```

### 차량 상세
```yaml
screen:
  id: SCR-VEH-002
  name: 차량 상세
  service: TrackCar Mobile
  domain: 차량
  route: /vehicles/:vehicleId
  menu_path: Vehicles > 차량 상세
  roles: [OWNER, STAFF]
  purpose: 선택 차량의 상태, 위치, 기사 연결, 알림, 운행 요약을 조회하고 OWNER는 일부 운영 정보를 수정한다.

layout:
  type: detail
  platform: mobile
  navigation:
    header: 뒤로가기 + 차량번호 + 더보기
    body: 상태 요약 + 지도 + 정보 섹션 + 최근 알림 + 최근 운행
    footer: 없음

data_scope:
  primary_entity: vehicle
  params: [vehicle_id, organization_id]
  filters: []
  sort: [alert.occurred_at desc, trip.started_at desc]
  pagination:
    enabled: false

components:
  - id: cmp_status_summary
    type: summary_card
    label: 차량 상태 요약
    required: false
    readonly: true
    visible: true
    enabled: true
    options: [vehicle_status, trip_status, install_status, assigned_status]
    data_binding: vehicle.summary
    description: 상태 기반 핵심 정보 상단 카드
  - id: cmp_last_received_at
    type: text
    label: 최종 수신 시각
    required: false
    readonly: true
    visible: true
    enabled: true
    options: []
    data_binding: vehicle.last_received_at
    description: 장치 데이터 최종 수신 시각
  - id: cmp_location_map
    type: map
    label: 현재 위치
    required: false
    readonly: true
    visible: true
    enabled: true
    options: [marker 1개, 위치 상태 표시]
    data_binding: vehicle.current_location
    description: 현재 위치와 좌표, 주소 표시
  - id: cmp_info_vehicle
    type: key_value
    label: 차량 정보
    required: false
    readonly: true
    visible: true
    enabled: true
    options: [차량번호, 자동차유형코드, VIN, 별칭, 그룹, 설치상태, 운행상태]
    data_binding: vehicle.info
    description: 기본 차량 정보 (vehicle_type: eTAS 전송용 자동차유형코드)
  - id: cmp_info_driver
    type: key_value
    label: 담당 기사
    required: false
    readonly: false
    visible: true
    enabled: true
    options: [기사명, 연락처, 운전자코드, 연결상태]
    data_binding: vehicle.driver_link
    description: OWNER는 변경 가능, STAFF는 읽기 전용 (driver_code: eTAS 전송용)
  - id: cmp_edit_driver_link
    type: button
    label: 담당 기사 변경
    required: false
    readonly: false
    visible: OWNER only
    enabled: true
    options: [bottom_sheet]
    data_binding: none
    description: 기사 연결 변경 바텀시트 진입
  - id: cmp_edit_group
    type: button
    label: 그룹 변경
    required: false
    readonly: false
    visible: OWNER only
    enabled: true
    options: [bottom_sheet]
    data_binding: none
    description: 그룹 변경 바텀시트 진입
  - id: cmp_recent_alerts
    type: list
    label: 최근 알림
    required: false
    readonly: true
    visible: true
    enabled: true
    options: [최근 5건]
    data_binding: vehicle.recent_alerts
    description: 최근 알림 내역
  - id: cmp_recent_trips
    type: list
    label: 최근 운행
    required: false
    readonly: true
    visible: true
    enabled: true
    options: [최근 3건]
    data_binding: vehicle.recent_trips
    description: 최근 운행 요약

actions:
  - trigger: 화면 진입
    target: 차량 상세
    behavior: 상세 정보와 최근 알림/운행 요약 조회
    api_ref: GET /v1/mobile/vehicles/{vehicleId}
    success_result: 상세 렌더링
    error_result: 오류 상태 표시
  - trigger: 담당 기사 변경 저장
    target: 차량 상세
    behavior: 차량-기사 연결 변경
    api_ref: PATCH /v1/mobile/vehicles/{vehicleId}/driver-link
    success_result: 담당 기사 정보 갱신, 토스트 표시
    error_result: 오류 토스트와 기존값 유지
  - trigger: 그룹 변경 저장
    target: 차량 상세
    behavior: 차량 소속 그룹 변경
    api_ref: PATCH /v1/mobile/vehicles/{vehicleId}/group
    success_result: 그룹 정보 갱신
    error_result: 오류 토스트 표시

states:
  initial: 스켈레톤 상세 레이아웃 표시
  normal: 상세 정보 정상 표시
  loading: 섹션 단위 로딩 가능
  empty: 최근 알림/운행 섹션별 이력이 없습니다.
  error: 차량 정보를 불러오지 못했습니다.
  no_permission: 해당 차량을 조회할 권한이 없습니다.
  conditional:
    - 담당 기사 미연결 시 경고 라벨 표시
    - 위치 정보 없음 시 지도 대신 안내문 표시
    - OWNER만 편집 버튼 노출

validation:
  driver_id:
    rule: 기사 변경 시 선택 필수
    message: 연결할 기사를 선택해 주세요.
  group_id:
    rule: 변경 시 필수
    message: 이동할 그룹을 선택해 주세요.

permissions:
  view: OWNER, STAFF
  create: none
  update: OWNER
  delete: none
  field_control:
    - OWNER: driver_link, group 수정 가능
    - STAFF: 전 항목 읽기 전용

api:
  method: GET/PATCH
  endpoint: /v1/mobile/vehicles/{vehicleId}
  purpose: 차량 상세 조회 및 그룹/기사 연결 변경
  request:
    get:
      path:
        vehicleId: string
      query:
        organization_id: string
    patch_driver_link:
      body:
        driver_id: string
    patch_group:
      body:
        group_id: string
  response:
    vehicle: object
    recent_alerts: array
    recent_trips: array

navigation_rules:
  entry_points: [차량 목록 카드 탭, 홈 조치 필요 차량 탭, 알림 상세에서 차량 보기]
  exits: [차량 목록, 알림 상세, 기사 선택 바텀시트, 그룹 선택 바텀시트]

ui_text:
  title: 차량 상세
  empty_message: 이력이 없습니다.
  error_message: 차량 정보를 불러오지 못했습니다.
  toast: [담당 기사가 변경되었습니다., 그룹이 변경되었습니다., 변경에 실패했습니다.]
  confirm_message: [담당 기사 연결을 해제할까요?]

test_points:
  - OWNER에서만 편집 버튼 노출 확인
  - 기사 연결 변경 후 목록/상세 상태 동기화 확인
  - 그룹 변경 권한 및 값 검증 확인
  - STAFF 권한에서 수정 API 호출 차단 확인
```

### 운행 이력 목록
```yaml
screen:
  id: SCR-TRIP-001
  name: 운행 이력 목록
  service: TrackCar Mobile
  domain: 운행
  route: /trips
  menu_path: Trips > 운행 이력
  roles: [OWNER, STAFF]
  purpose: 기간, 차량, 그룹 기준으로 운행 이력을 조회한다.

layout:
  type: list
  platform: mobile
  navigation:
    header: 타이틀 + 기간 선택 + 필터
    body: 요약 카드 + 운행 리스트
    footer: 하단 탭바

data_scope:
  primary_entity: trip
  params: [organization_id, from_date, to_date, vehicle_id, group_id, trip_status, page, size]
  filters: [from_date, to_date, vehicle_id, group_id, trip_status]
  sort: [started_at desc]
  pagination:
    enabled: true
    type: pagination

components:
  - id: cmp_date_range
    type: date_range_picker
    label: 조회 기간
    required: true
    readonly: false
    visible: true
    enabled: true
    options: 최근 7일 기본
    data_binding: query.from_date, query.to_date
    description: 기간 변경 시 재조회
  - id: cmp_filter_button
    type: button
    label: 조건 필터
    required: false
    readonly: false
    visible: true
    enabled: true
    options: [bottom_sheet]
    data_binding: none
    description: 차량/그룹/상태 필터 열기
  - id: cmp_trip_summary
    type: summary_card
    label: 기간 요약
    required: false
    readonly: true
    visible: true
    enabled: true
    options: [총운행횟수, 총거리, 총시간]
    data_binding: trips.summary
    description: 조회 기간 전체 요약
  - id: cmp_trip_list
    type: list
    label: 운행 이력 리스트
    required: false
    readonly: true
    visible: true
    enabled: true
    options: [차량, 기사, 시작/종료 시각, 거리, 시간, 상태]
    data_binding: trips.items
    description: 운행 이력 목록 표시

actions:
  - trigger: 화면 진입
    target: 운행 이력 목록
    behavior: 기본 기간 최근 7일로 조회
    api_ref: GET /v1/mobile/trips
    success_result: 목록과 요약 렌더링
    error_result: 오류 상태 표시
  - trigger: 기간 변경
    target: 운행 이력 목록
    behavior: 새 기간으로 재조회
    api_ref: GET /v1/mobile/trips
    success_result: 목록 갱신
    error_result: 토스트 '운행 이력을 불러오지 못했습니다.'
  - trigger: 이력 항목 탭
    target: 운행 상세
    behavior: 상세 화면 이동
    api_ref: none
    success_result: 상세 진입
    error_result: none

states:
  initial: 스켈레톤 리스트 표시
  normal: 기간 요약과 이력 목록 표시
  loading: 조회 조건 변경 시 로딩 표시
  empty: 조회 기간에 운행 이력이 없습니다.
  error: 운행 이력을 불러오지 못했습니다.
  no_permission: 운행 이력을 조회할 권한이 없습니다.
  conditional:
    - 차량 선택 시 요약을 해당 차량 기준으로 재계산
    - STAFF는 허용 차량만 필터 옵션 노출

validation:
  from_date:
    rule: 필수, to_date 이하
    message: 시작일을 확인해 주세요.
  to_date:
    rule: 필수, from_date 이상, 최대 조회기간 93일
    message: 종료일을 확인해 주세요.

permissions:
  view: OWNER, STAFF
  create: none
  update: none
  delete: none
  field_control:
    - STAFF는 권한 없는 차량 필터 숨김

api:
  method: GET
  endpoint: /v1/mobile/trips
  purpose: 운행 이력 목록 조회
  request:
    query:
      organization_id: string
      from_date: date
      to_date: date
      vehicle_id: string|null
      group_id: string|null
      trip_status: string|null
      page: number
      size: number
  response:
    summary:
      trip_count: number
      total_distance_km: number
      total_duration_min: number
    items: array
    page: number
    total: number

navigation_rules:
  entry_points: [하단 탭 Trips, 차량 상세 최근 운행 항목 탭]
  exits: [운행 상세, 필터 바텀시트, 홈]

ui_text:
  title: 운행 이력
  empty_message: 조회 기간에 운행 이력이 없습니다.
  error_message: 운행 이력을 불러오지 못했습니다.
  toast: [조회 조건이 적용되었습니다., 운행 이력을 불러오지 못했습니다.]
  confirm_message: []

test_points:
  - 최근 7일 기본값 적용 확인
  - 최대 조회기간 93일 검증 확인
  - 요약 카드 수치와 목록 합계 일치 확인
```

### 운행 상세
```yaml
screen:
  id: SCR-TRIP-002
  name: 운행 상세
  service: TrackCar Mobile
  domain: 운행
  route: /trips/:tripId
  menu_path: Trips > 운행 상세
  roles: [OWNER, STAFF]
  purpose: 개별 운행 이력의 경로, 시간, 거리, 이벤트를 조회한다.

layout:
  type: detail
  platform: mobile
  navigation:
    header: 뒤로가기 + 운행 상세
    body: 요약 카드 + 지도 + 이벤트 타임라인
    footer: 없음

data_scope:
  primary_entity: trip
  params: [trip_id, organization_id]
  filters: []
  sort: [event_time asc]
  pagination:
    enabled: false

components:
  - id: cmp_trip_summary
    type: summary_card
    label: 운행 요약
    required: false
    readonly: true
    visible: true
    enabled: true
    options: [차량, 기사, 시작/종료, 거리, 시간]
    data_binding: trip.summary
    description: 운행 핵심 정보 표시
  - id: cmp_route_map
    type: map
    label: 이동 경로
    required: false
    readonly: true
    visible: true
    enabled: true
    options: [start/end marker, polyline]
    data_binding: trip.route_points
    description: 운행 경로 시각화
  - id: cmp_event_timeline
    type: timeline
    label: 운행 이벤트
    required: false
    readonly: true
    visible: true
    enabled: true
    options: [시동ON, 시동OFF, 정차, 알림 이벤트]
    data_binding: trip.events
    description: 시간순 이벤트 나열

actions:
  - trigger: 화면 진입
    target: 운행 상세
    behavior: 운행 상세 조회
    api_ref: GET /v1/mobile/trips/{tripId}
    success_result: 상세 렌더링
    error_result: 오류 상태 표시

states:
  initial: 스켈레톤 상세 표시
  normal: 지도와 이벤트 표시
  loading: 지도 데이터 로딩 인디케이터 표시
  empty: 운행 이벤트 정보가 없습니다.
  error: 운행 상세를 불러오지 못했습니다.
  no_permission: 해당 운행 이력을 조회할 권한이 없습니다.
  conditional:
    - 경로 포인트 없으면 지도 대신 안내문 표시

validation:
  trip_id:
    rule: 필수
    message: 올바르지 않은 운행 정보입니다.

permissions:
  view: OWNER, STAFF
  create: none
  update: none
  delete: none
  field_control: []

api:
  method: GET
  endpoint: /v1/mobile/trips/{tripId}
  purpose: 운행 상세 조회
  request:
    path:
      tripId: string
    query:
      organization_id: string
  response:
    summary: object
    route_points: array
    events: array

navigation_rules:
  entry_points: [운행 이력 목록 항목 탭, 차량 상세 최근 운행 탭]
  exits: [운행 이력 목록, 차량 상세]

ui_text:
  title: 운행 상세
  empty_message: 운행 이벤트 정보가 없습니다.
  error_message: 운행 상세를 불러오지 못했습니다.
  toast: []
  confirm_message: []

test_points:
  - 경로 포인트 없는 운행 데이터 예외 처리 확인
  - 타임라인 이벤트 오름차순 정렬 확인
  - 권한 없는 운행 접근 차단 확인
```

### 알림 센터
```yaml
screen:
  id: SCR-ALT-001
  name: 알림 센터
  service: TrackCar Mobile
  domain: 알림
  route: /alerts
  menu_path: Alerts
  roles: [OWNER, STAFF]
  purpose: 차량 이상, 통신, 위치, 운영 알림을 조회하고 상태를 확인한다.

layout:
  type: list
  platform: mobile
  navigation:
    header: 타이틀 + 심각도 필터 + 상태 필터
    body: 알림 리스트
    footer: 하단 탭바

data_scope:
  primary_entity: alert
  params: [organization_id, severity, status, vehicle_id, group_id, page, size]
  filters: [severity, status, vehicle_id, group_id, alert_type]
  sort: [occurred_at desc]
  pagination:
    enabled: true
    type: infinite_scroll

components:
  - id: cmp_severity_filter
    type: chip_filter
    label: 심각도
    required: false
    readonly: false
    visible: true
    enabled: true
    options: [전체, 심각, 높음, 보통, 낮음]
    data_binding: query.severity
    description: 심각도 기준 필터
  - id: cmp_status_filter
    type: chip_filter
    label: 처리 상태
    required: false
    readonly: false
    visible: true
    enabled: true
    options: [전체, 미확인, 확인완료]
    data_binding: query.status
    description: 알림 확인 상태 필터
  - id: cmp_alert_list
    type: list
    label: 알림 목록
    required: false
    readonly: true
    visible: true
    enabled: true
    options: [severity badge, 제목, 차량, 발생시각, 확인상태]
    data_binding: alerts.items
    description: 최신순 알림 나열
  - id: cmp_bulk_mark_read
    type: button
    label: 모두 확인 처리
    required: false
    readonly: false
    visible: OWNER only
    enabled: true
    options: []
    data_binding: none
    description: 현재 필터 범위의 미확인 알림을 일괄 확인 처리

actions:
  - trigger: 화면 진입
    target: 알림 센터
    behavior: 최근 알림 조회
    api_ref: GET /v1/mobile/alerts
    success_result: 목록 렌더링
    error_result: 오류 상태 표시
  - trigger: 알림 탭
    target: 알림 상세
    behavior: 상세 이동, 미확인 알림이면 읽음 처리 예약
    api_ref: GET /v1/mobile/alerts/{alertId}
    success_result: 상세 진입
    error_result: none
  - trigger: 모두 확인 처리
    target: 알림 센터
    behavior: 현재 조건 미확인 알림 확인 처리
    api_ref: PATCH /v1/mobile/alerts/read-all
    success_result: 목록 상태 갱신
    error_result: 토스트 '알림 상태를 변경하지 못했습니다.'

states:
  initial: 스켈레톤 리스트 표시
  normal: 알림 목록 표시
  loading: 첫 로딩 및 하단 추가 로딩 표시
  empty: 표시할 알림이 없습니다.
  error: 알림을 불러오지 못했습니다.
  no_permission: 알림을 조회할 권한이 없습니다.
  conditional:
    - STAFF는 모두 확인 처리 버튼 숨김
    - severity가 CRITICAL이면 카드 강조 표시

validation:
  page:
    rule: 1 이상
    message: 올바르지 않은 페이지입니다.

permissions:
  view: OWNER, STAFF
  create: none
  update: OWNER, STAFF
  delete: none
  field_control:
    - OWNER: read-all 사용 가능
    - STAFF: 개별 알림 확인만 가능

api:
  method: GET/PATCH
  endpoint: /v1/mobile/alerts
  purpose: 알림 목록 조회 및 확인 상태 변경
  request:
    get:
      query:
        organization_id: string
        severity: string|null
        status: string|null
        vehicle_id: string|null
        group_id: string|null
        page: number
        size: number
    patch_read_all:
      body:
        organization_id: string
        severity: string|null
        status: UNREAD
  response:
    items: array
    total_unread: number

navigation_rules:
  entry_points: [하단 탭 Alerts, 홈 최근 알림, 푸시 알림 탭]
  exits: [알림 상세, 차량 상세, 홈]

ui_text:
  title: 알림 센터
  empty_message: 표시할 알림이 없습니다.
  error_message: 알림을 불러오지 못했습니다.
  toast: [알림 상태가 변경되었습니다., 알림 상태를 변경하지 못했습니다.]
  confirm_message: [현재 목록의 미확인 알림을 모두 확인 처리할까요?]

test_points:
  - severity/status 복합 필터 확인
  - 푸시 알림 딥링크 진입 확인
  - OWNER만 일괄 확인 처리 노출 확인
```

### 알림 상세
```yaml
screen:
  id: SCR-ALT-002
  name: 알림 상세
  service: TrackCar Mobile
  domain: 알림
  route: /alerts/:alertId
  menu_path: Alerts > 알림 상세
  roles: [OWNER, STAFF]
  purpose: 알림 원인, 상태, 차량 정보, 조치 힌트를 상세 조회한다.

layout:
  type: detail
  platform: mobile
  navigation:
    header: 뒤로가기 + 알림 상세
    body: 심각도 카드 + 본문 + 관련 차량 정보 + 조치 버튼
    footer: 없음

data_scope:
  primary_entity: alert
  params: [alert_id, organization_id]
  filters: []
  sort: []
  pagination:
    enabled: false

components:
  - id: cmp_alert_summary
    type: summary_card
    label: 알림 요약
    required: false
    readonly: true
    visible: true
    enabled: true
    options: [severity, title, status, occurred_at]
    data_binding: alert.summary
    description: 알림 핵심 정보 표시
  - id: cmp_alert_body
    type: text_block
    label: 알림 내용
    required: false
    readonly: true
    visible: true
    enabled: true
    options: []
    data_binding: alert.description
    description: 발생 원인, 감지 조건, 권장 조치
  - id: cmp_related_vehicle
    type: card
    label: 관련 차량
    required: false
    readonly: true
    visible: true
    enabled: true
    options: [차량번호, 상태, 최종수신시각]
    data_binding: alert.vehicle
    description: 관련 차량 정보 표시
  - id: cmp_mark_read
    type: button
    label: 확인 완료
    required: false
    readonly: false
    visible: true
    enabled: alert.status == UNREAD
    options: []
    data_binding: none
    description: 알림을 확인 완료 처리
  - id: cmp_go_vehicle
    type: button
    label: 차량 상세 보기
    required: false
    readonly: false
    visible: true
    enabled: true
    options: []
    data_binding: alert.vehicle.vehicle_id
    description: 차량 상세로 이동

actions:
  - trigger: 화면 진입
    target: 알림 상세
    behavior: 상세 조회와 미확인 알림 자동 확인 처리(optional) 수행
    api_ref: GET /v1/mobile/alerts/{alertId}
    success_result: 상세 렌더링
    error_result: 오류 상태 표시
  - trigger: 확인 완료 버튼 탭
    target: 알림 상세
    behavior: 알림 상태를 확인 완료로 변경
    api_ref: PATCH /v1/mobile/alerts/{alertId}/read
    success_result: 상태 갱신, 토스트 표시
    error_result: 토스트 표시

states:
  initial: 스켈레톤 표시
  normal: 상세 표시
  loading: 버튼 단위 로딩 가능
  empty: 해당 없음
  error: 알림 상세를 불러오지 못했습니다.
  no_permission: 해당 알림을 조회할 권한이 없습니다.
  conditional:
    - 이미 확인 완료된 알림은 버튼 비노출
    - 관련 차량 없음 시 차량 카드 숨김

validation:
  alert_id:
    rule: 필수
    message: 올바르지 않은 알림 정보입니다.

permissions:
  view: OWNER, STAFF
  create: none
  update: OWNER, STAFF
  delete: none
  field_control: []

api:
  method: GET/PATCH
  endpoint: /v1/mobile/alerts/{alertId}
  purpose: 알림 상세 조회 및 확인 처리
  request:
    path:
      alertId: string
  response:
    alert: object

navigation_rules:
  entry_points: [알림 센터 목록 탭, 홈 최근 알림 탭, 푸시 알림 딥링크]
  exits: [알림 센터, 차량 상세]

ui_text:
  title: 알림 상세
  empty_message: ''
  error_message: 알림 상세를 불러오지 못했습니다.
  toast: [알림을 확인 완료했습니다., 알림 상태를 변경하지 못했습니다.]
  confirm_message: []

test_points:
  - 미확인 알림 진입 시 자동 확인 처리 정책 반영 여부 확인
  - 확인 완료 버튼 중복 탭 방지 확인
  - 관련 차량 없는 알림 예외 처리 확인
```

### 워크스페이스 홈
```yaml
screen:
  id: SCR-WS-001
  name: 워크스페이스 홈
  service: TrackCar Mobile
  domain: 조직운영
  route: /workspace
  menu_path: Workspace
  roles: [OWNER]
  purpose: OWNER가 담당자, 그룹, 차량-기사 연결 운영 메뉴에 진입한다.

layout:
  type: menu
  platform: mobile
  navigation:
    header: 타이틀
    body: 운영 카드 메뉴
    footer: 하단 탭바

data_scope:
  primary_entity: workspace_summary
  params: [organization_id]
  filters: []
  sort: []
  pagination:
    enabled: false

components:
  - id: cmp_staff_menu
    type: menu_card
    label: 담당자 관리
    required: false
    readonly: false
    visible: true
    enabled: true
    options: 요약 수치 포함
    data_binding: workspace.staff_summary
    description: 담당자 목록/등록 이동
  - id: cmp_group_menu
    type: menu_card
    label: 그룹 관리
    required: false
    readonly: false
    visible: true
    enabled: true
    options: 그룹 수 표시
    data_binding: workspace.group_summary
    description: 그룹 목록/등록 이동
  - id: cmp_link_menu
    type: menu_card
    label: 차량-기사 연결 관리
    required: false
    readonly: false
    visible: true
    enabled: true
    options: 미연결 차량 수 표시
    data_binding: workspace.link_summary
    description: 차량-기사 상시 연결 관리 이동

actions:
  - trigger: 화면 진입
    target: 워크스페이스 홈
    behavior: 운영 요약 정보 조회
    api_ref: GET /v1/mobile/workspace
    success_result: 메뉴 카드 수치 표시
    error_result: 오류 상태 표시

states:
  initial: 스켈레톤 카드 표시
  normal: 메뉴 카드 표시
  loading: 로딩 인디케이터 표시
  empty: 요약 데이터가 없어도 메뉴는 표시
  error: 워크스페이스 정보를 불러오지 못했습니다.
  no_permission: 워크스페이스 접근 권한이 없습니다.
  conditional: []

validation: {}

permissions:
  view: OWNER
  create: OWNER
  update: OWNER
  delete: none
  field_control: []

api:
  method: GET
  endpoint: /v1/mobile/workspace
  purpose: 워크스페이스 요약 조회
  request:
    query:
      organization_id: string
  response:
    staff_summary: object
    group_summary: object
    link_summary: object

navigation_rules:
  entry_points: [하단 탭 Workspace, 홈 빠른 이동]
  exits: [담당자 목록, 그룹 목록, 차량-기사 연결 관리]

ui_text:
  title: 워크스페이스
  empty_message: ''
  error_message: 워크스페이스 정보를 불러오지 못했습니다.
  toast: []
  confirm_message: []

test_points:
  - OWNER 외 권한 차단 확인
  - 요약 수치 링크 동작 확인
```

### 담당자 목록
```yaml
screen:
  id: SCR-STAFF-001
  name: 담당자 목록
  service: TrackCar Mobile
  domain: 조직운영
  route: /workspace/staff
  menu_path: Workspace > 담당자 관리
  roles: [OWNER]
  purpose: 담당자 계정을 조회하고 등록/비활성화/권한 범위를 관리한다.

layout:
  type: list
  platform: mobile
  navigation:
    header: 타이틀 + 등록 버튼
    body: 검색 + 담당자 리스트
    footer: 없음

data_scope:
  primary_entity: staff_user
  params: [organization_id, keyword, status, page, size]
  filters: [status]
  sort: [created_at desc]
  pagination:
    enabled: true
    type: pagination

components:
  - id: cmp_add_staff
    type: button
    label: 담당자 추가
    required: false
    readonly: false
    visible: true
    enabled: true
    options: []
    data_binding: none
    description: 담당자 등록 화면 이동
  - id: cmp_search
    type: search
    label: 이름/이메일 검색
    required: false
    readonly: false
    visible: true
    enabled: true
    options: debounce 500ms
    data_binding: query.keyword
    description: 담당자 검색
  - id: cmp_status_filter
    type: chip_filter
    label: 상태
    required: false
    readonly: false
    visible: true
    enabled: true
    options: [전체, 활성, 비활성]
    data_binding: query.status
    description: 담당자 상태 필터
  - id: cmp_staff_list
    type: list
    label: 담당자 리스트
    required: false
    readonly: true
    visible: true
    enabled: true
    options: [이름, 이메일, 상태, 알림수신, 접근범위]
    data_binding: staff.items
    description: 담당자 카드 리스트

actions:
  - trigger: 화면 진입
    target: 담당자 목록
    behavior: 목록 조회
    api_ref: GET /v1/mobile/staff-users
    success_result: 리스트 표시
    error_result: 오류 상태 표시
  - trigger: 담당자 추가 버튼 탭
    target: 담당자 등록
    behavior: 등록 화면 이동
    api_ref: none
    success_result: 등록 화면 진입
    error_result: none
  - trigger: 담당자 카드 탭
    target: 담당자 상세
    behavior: 상세 화면 이동
    api_ref: none
    success_result: 상세 진입
    error_result: none

states:
  initial: 스켈레톤 리스트 표시
  normal: 담당자 목록 표시
  loading: 조회 중 로딩 표시
  empty: 등록된 담당자가 없습니다.
  error: 담당자 목록을 불러오지 못했습니다.
  no_permission: 담당자 관리 권한이 없습니다.
  conditional: []

validation:
  keyword:
    rule: 최대 50자
    message: 검색어는 50자 이하로 입력해 주세요.

permissions:
  view: OWNER
  create: OWNER
  update: OWNER
  delete: OWNER
  field_control: []

api:
  method: GET
  endpoint: /v1/mobile/staff-users
  purpose: 담당자 목록 조회
  request:
    query:
      organization_id: string
      keyword: string|null
      status: string|null
      page: number
      size: number
  response:
    items: array
    page: number
    total: number

navigation_rules:
  entry_points: [워크스페이스 홈]
  exits: [담당자 등록, 담당자 상세, 워크스페이스 홈]

ui_text:
  title: 담당자 관리
  empty_message: 등록된 담당자가 없습니다.
  error_message: 담당자 목록을 불러오지 못했습니다.
  toast: []
  confirm_message: []

test_points:
  - 담당자 검색/상태 필터 확인
  - 빈 목록 상태에서 등록 유도 확인
```

### 담당자 등록
```yaml
screen:
  id: SCR-STAFF-002
  name: 담당자 등록
  service: TrackCar Mobile
  domain: 조직운영
  route: /workspace/staff/new
  menu_path: Workspace > 담당자 관리 > 담당자 등록
  roles: [OWNER]
  purpose: 신규 담당자 계정을 생성하고 알림 수신 및 접근 범위를 설정한다.

layout:
  type: form
  platform: mobile
  navigation:
    header: 뒤로가기 + 저장
    body: 기본정보 + 알림설정 + 접근범위
    footer: 하단 고정 저장 버튼

data_scope:
  primary_entity: staff_user
  params: [organization_id]
  filters: []
  sort: []
  pagination:
    enabled: false

components:
  - id: fld_name
    type: text_input
    label: 담당자 이름
    required: true
    readonly: false
    visible: true
    enabled: true
    options: maxLength 30
    data_binding: form.name
    description: 담당자 실명
  - id: fld_email
    type: email_input
    label: 이메일
    required: true
    readonly: false
    visible: true
    enabled: true
    options: unique
    data_binding: form.email
    description: 로그인 계정 이메일
  - id: fld_phone
    type: phone_input
    label: 휴대폰 번호
    required: true
    readonly: false
    visible: true
    enabled: true
    options: KR format
    data_binding: form.phone
    description: 연락처
  - id: fld_notification
    type: checkbox_group
    label: 알림 수신
    required: false
    readonly: false
    visible: true
    enabled: true
    options: [푸시수신, 이메일수신]
    data_binding: form.notification_channels
    description: 담당자 알림 채널 설정
  - id: fld_scope_type
    type: radio
    label: 접근 범위
    required: true
    readonly: false
    visible: true
    enabled: true
    options: [전체 그룹, 선택 그룹]
    data_binding: form.scope_type
    description: 담당자 조회 범위 설정
  - id: fld_group_ids
    type: multi_select
    label: 허용 그룹
    required: conditional
    readonly: false
    visible: form.scope_type == SELECTED_GROUPS
    enabled: true
    options: 조직 그룹 목록
    data_binding: form.group_ids
    description: 선택 그룹 모드일 때 필수

actions:
  - trigger: 저장 버튼 탭
    target: 담당자 등록
    behavior: 입력값 검증 후 계정 생성
    api_ref: POST /v1/mobile/staff-users
    success_result: 목록 복귀 후 생성 토스트 표시
    error_result: 필드 오류 또는 일반 오류 표시

states:
  initial: 빈 폼 표시
  normal: 입력 가능 상태
  loading: 저장 버튼 로딩 표시
  empty: 해당 없음
  error: 상단 오류 배너 표시 가능
  no_permission: 담당자 등록 권한이 없습니다.
  conditional:
    - 선택 그룹 모드일 때 허용 그룹 필드 표시

validation:
  name:
    rule: 필수, 2~30자
    message: 담당자 이름을 2자 이상 30자 이하로 입력해 주세요.
  email:
    rule: 필수, 이메일 형식, 중복 불가
    message: 올바른 이메일을 입력해 주세요.
  phone:
    rule: 필수, 휴대폰 형식
    message: 휴대폰 번호를 정확히 입력해 주세요.
  scope_type:
    rule: 필수
    message: 접근 범위를 선택해 주세요.
  group_ids:
    rule: 선택 그룹 모드일 때 1개 이상 필수
    message: 최소 1개 그룹을 선택해 주세요.

permissions:
  view: OWNER
  create: OWNER
  update: none
  delete: none
  field_control: []

api:
  method: POST
  endpoint: /v1/mobile/staff-users
  purpose: 담당자 계정 생성
  request:
    body:
      organization_id: string
      name: string
      email: string
      phone: string
      notification_channels: array
      scope_type: string
      group_ids: array
  response:
    staff_user_id: string
    invitation_status: string

navigation_rules:
  entry_points: [담당자 목록 등록 버튼]
  exits: [담당자 목록, 뒤로가기]

ui_text:
  title: 담당자 추가
  empty_message: ''
  error_message: 담당자 계정을 생성하지 못했습니다.
  toast: [담당자 계정이 생성되었습니다., 담당자 계정을 생성하지 못했습니다.]
  confirm_message: [저장하지 않고 나갈까요?]

test_points:
  - 이메일 중복 검증 확인
  - 선택 그룹 모드 조건부 validation 확인
  - 저장 후 목록 복귀 및 토스트 확인
```

### 담당자 상세
```yaml
screen:
  id: SCR-STAFF-003
  name: 담당자 상세
  service: TrackCar Mobile
  domain: 조직운영
  route: /workspace/staff/:staffUserId
  menu_path: Workspace > 담당자 관리 > 담당자 상세
  roles: [OWNER]
  purpose: 담당자 상태, 알림 설정, 접근 범위를 조회하고 수정/비활성화한다.

layout:
  type: detail_form
  platform: mobile
  navigation:
    header: 뒤로가기 + 저장
    body: 기본정보 + 알림설정 + 접근범위 + 상태관리
    footer: 하단 저장/비활성화 버튼

data_scope:
  primary_entity: staff_user
  params: [staff_user_id, organization_id]
  filters: []
  sort: []
  pagination:
    enabled: false

components:
  - id: fld_name
    type: text_input
    label: 담당자 이름
    required: true
    readonly: false
    visible: true
    enabled: true
    options: maxLength 30
    data_binding: form.name
    description: 수정 가능
  - id: fld_email
    type: email_input
    label: 이메일
    required: true
    readonly: true
    visible: true
    enabled: false
    options: []
    data_binding: form.email
    description: 계정 식별자로 수정 불가
  - id: fld_phone
    type: phone_input
    label: 휴대폰 번호
    required: true
    readonly: false
    visible: true
    enabled: true
    options: KR format
    data_binding: form.phone
    description: 수정 가능
  - id: fld_notification
    type: checkbox_group
    label: 알림 수신
    required: false
    readonly: false
    visible: true
    enabled: true
    options: [푸시수신, 이메일수신]
    data_binding: form.notification_channels
    description: 수정 가능
  - id: fld_scope_type
    type: radio
    label: 접근 범위
    required: true
    readonly: false
    visible: true
    enabled: true
    options: [전체 그룹, 선택 그룹]
    data_binding: form.scope_type
    description: 수정 가능
  - id: fld_group_ids
    type: multi_select
    label: 허용 그룹
    required: conditional
    readonly: false
    visible: form.scope_type == SELECTED_GROUPS
    enabled: true
    options: 조직 그룹 목록
    data_binding: form.group_ids
    description: 수정 가능
  - id: fld_status
    type: status_badge
    label: 계정 상태
    required: false
    readonly: true
    visible: true
    enabled: true
    options: [활성, 비활성]
    data_binding: form.status
    description: 읽기 전용
  - id: btn_deactivate
    type: button
    label: 계정 비활성화
    required: false
    readonly: false
    visible: form.status == ACTIVE
    enabled: true
    options: danger
    data_binding: none
    description: 비활성화 처리
  - id: btn_activate
    type: button
    label: 계정 재활성화
    required: false
    readonly: false
    visible: form.status == INACTIVE
    enabled: true
    options: secondary
    data_binding: none
    description: 재활성화 처리

actions:
  - trigger: 화면 진입
    target: 담당자 상세
    behavior: 상세 정보 조회
    api_ref: GET /v1/mobile/staff-users/{staffUserId}
    success_result: 폼 표시
    error_result: 오류 상태 표시
  - trigger: 저장 버튼 탭
    target: 담당자 상세
    behavior: 수정 내용 저장
    api_ref: PATCH /v1/mobile/staff-users/{staffUserId}
    success_result: 상세 값 갱신, 토스트 표시
    error_result: 필드 오류/일반 오류 표시
  - trigger: 계정 비활성화/재활성화 탭
    target: 담당자 상세
    behavior: 계정 상태 변경
    api_ref: PATCH /v1/mobile/staff-users/{staffUserId}/status
    success_result: 상태 배지 갱신
    error_result: 오류 토스트 표시

states:
  initial: 스켈레톤 폼 표시
  normal: 조회/수정 가능
  loading: 저장 또는 상태변경 로딩 표시
  empty: 해당 없음
  error: 담당자 정보를 불러오지 못했습니다.
  no_permission: 담당자 관리 권한이 없습니다.
  conditional:
    - 상태에 따라 활성/비활성 버튼 교차 노출

validation:
  name:
    rule: 필수, 2~30자
    message: 담당자 이름을 2자 이상 30자 이하로 입력해 주세요.
  phone:
    rule: 필수, 휴대폰 형식
    message: 휴대폰 번호를 정확히 입력해 주세요.
  group_ids:
    rule: 선택 그룹 모드일 때 1개 이상 필수
    message: 최소 1개 그룹을 선택해 주세요.

permissions:
  view: OWNER
  create: none
  update: OWNER
  delete: OWNER
  field_control:
    - email: readonly
    - status: system managed

api:
  method: GET/PATCH
  endpoint: /v1/mobile/staff-users/{staffUserId}
  purpose: 담당자 상세 조회, 수정, 상태 변경
  request:
    get:
      path:
        staffUserId: string
    patch:
      body:
        name: string
        phone: string
        notification_channels: array
        scope_type: string
        group_ids: array
    patch_status:
      body:
        status: ACTIVE|INACTIVE
  response:
    staff_user: object

navigation_rules:
  entry_points: [담당자 목록 카드 탭]
  exits: [담당자 목록, 뒤로가기]

ui_text:
  title: 담당자 상세
  empty_message: ''
  error_message: 담당자 정보를 불러오지 못했습니다.
  toast: [담당자 정보가 저장되었습니다., 계정 상태가 변경되었습니다., 저장하지 못했습니다.]
  confirm_message: [이 담당자 계정을 비활성화할까요?, 저장하지 않고 나갈까요?]

test_points:
  - 이메일 readonly 확인
  - 상태변경 버튼 조건부 노출 확인
  - group_ids 조건부 validation 확인
```

### 그룹 목록
```yaml
screen:
  id: SCR-GRP-001
  name: 그룹 목록
  service: TrackCar Mobile
  domain: 조직운영
  route: /workspace/groups
  menu_path: Workspace > 그룹 관리
  roles: [OWNER]
  purpose: 그룹을 조회하고 등록/수정/삭제한다.

layout:
  type: list
  platform: mobile
  navigation:
    header: 타이틀 + 등록 버튼
    body: 그룹 리스트
    footer: 없음

data_scope:
  primary_entity: group
  params: [organization_id]
  filters: []
  sort: [created_at asc]
  pagination:
    enabled: false

components:
  - id: cmp_add_group
    type: button
    label: 그룹 추가
    required: false
    readonly: false
    visible: true
    enabled: true
    options: []
    data_binding: none
    description: 그룹 등록 화면 이동
  - id: cmp_group_list
    type: list
    label: 그룹 리스트
    required: false
    readonly: true
    visible: true
    enabled: true
    options: [그룹명, 설명, 차량수, 담당자수]
    data_binding: groups.items
    description: 그룹 카드 리스트

actions:
  - trigger: 화면 진입
    target: 그룹 목록
    behavior: 그룹 목록 조회
    api_ref: GET /v1/mobile/groups
    success_result: 목록 렌더링
    error_result: 오류 상태 표시
  - trigger: 그룹 추가 버튼 탭
    target: 그룹 등록
    behavior: 등록 화면 이동
    api_ref: none
    success_result: 등록 화면 진입
    error_result: none
  - trigger: 그룹 카드 탭
    target: 그룹 상세
    behavior: 그룹 상세 이동
    api_ref: none
    success_result: 상세 진입
    error_result: none

states:
  initial: 스켈레톤 리스트 표시
  normal: 그룹 목록 표시
  loading: 로딩 인디케이터 표시
  empty: 등록된 그룹이 없습니다.
  error: 그룹 목록을 불러오지 못했습니다.
  no_permission: 그룹 관리 권한이 없습니다.
  conditional: []

validation: {}

permissions:
  view: OWNER
  create: OWNER
  update: OWNER
  delete: OWNER
  field_control: []

api:
  method: GET
  endpoint: /v1/mobile/groups
  purpose: 그룹 목록 조회
  request:
    query:
      organization_id: string
  response:
    items: array

navigation_rules:
  entry_points: [워크스페이스 홈]
  exits: [그룹 등록, 그룹 상세]

ui_text:
  title: 그룹 관리
  empty_message: 등록된 그룹이 없습니다.
  error_message: 그룹 목록을 불러오지 못했습니다.
  toast: []
  confirm_message: []

test_points:
  - 그룹 없는 상태 확인
  - 그룹 카드 수치 표시 정확성 확인
```

### 그룹 등록/상세
```yaml
screen:
  id: SCR-GRP-002
  name: 그룹 등록/상세
  service: TrackCar Mobile
  domain: 조직운영
  route: /workspace/groups/:groupId?
  menu_path: Workspace > 그룹 관리 > 그룹 등록/상세
  roles: [OWNER]
  purpose: 그룹을 등록하거나 상세 조회 후 수정/삭제한다.

layout:
  type: form
  platform: mobile
  navigation:
    header: 뒤로가기 + 저장
    body: 기본정보 + 소속 차량 요약
    footer: 하단 저장 버튼

data_scope:
  primary_entity: group
  params: [organization_id, group_id]
  filters: []
  sort: []
  pagination:
    enabled: false

components:
  - id: fld_group_name
    type: text_input
    label: 그룹명
    required: true
    readonly: false
    visible: true
    enabled: true
    options: maxLength 30, unique in organization
    data_binding: form.name
    description: 그룹명
  - id: fld_group_description
    type: textarea
    label: 설명
    required: false
    readonly: false
    visible: true
    enabled: true
    options: maxLength 200
    data_binding: form.description
    description: 그룹 설명
  - id: cmp_vehicle_count
    type: text
    label: 소속 차량 수
    required: false
    readonly: true
    visible: form.group_id exists
    enabled: true
    options: []
    data_binding: form.vehicle_count
    description: 상세 모드에서만 표시
  - id: btn_delete_group
    type: button
    label: 그룹 삭제
    required: false
    readonly: false
    visible: form.group_id exists
    enabled: form.vehicle_count == 0
    options: danger
    data_binding: none
    description: 소속 차량이 없을 때만 삭제 가능

actions:
  - trigger: 등록 저장
    target: 그룹 등록/상세
    behavior: 신규 그룹 생성
    api_ref: POST /v1/mobile/groups
    success_result: 목록 복귀 및 생성 토스트
    error_result: 필드 오류/일반 오류 표시
  - trigger: 상세 저장
    target: 그룹 등록/상세
    behavior: 그룹 정보 수정
    api_ref: PATCH /v1/mobile/groups/{groupId}
    success_result: 상세 갱신
    error_result: 오류 표시
  - trigger: 그룹 삭제
    target: 그룹 등록/상세
    behavior: 그룹 삭제
    api_ref: DELETE /v1/mobile/groups/{groupId}
    success_result: 목록 복귀 및 삭제 토스트
    error_result: 삭제 실패 토스트

states:
  initial: 등록 모드 빈 폼 또는 상세 모드 스켈레톤
  normal: 입력/조회 가능
  loading: 저장/삭제 로딩 표시
  empty: 해당 없음
  error: 그룹 정보를 처리하지 못했습니다.
  no_permission: 그룹 관리 권한이 없습니다.
  conditional:
    - 소속 차량 수 > 0이면 삭제 버튼 비활성화 및 안내문 노출

validation:
  name:
    rule: 필수, 2~30자, 조직 내 중복 불가
    message: 그룹명을 2자 이상 30자 이하로 입력해 주세요.
  description:
    rule: 200자 이하
    message: 설명은 200자 이하로 입력해 주세요.

permissions:
  view: OWNER
  create: OWNER
  update: OWNER
  delete: OWNER
  field_control:
    - vehicle_count: readonly

api:
  method: POST/PATCH/DELETE
  endpoint: /v1/mobile/groups
  purpose: 그룹 생성, 수정, 삭제
  request:
    post:
      body:
        organization_id: string
        name: string
        description: string|null
    patch:
      body:
        name: string
        description: string|null
  response:
    group: object

navigation_rules:
  entry_points: [그룹 목록 등록 버튼, 그룹 목록 카드 탭]
  exits: [그룹 목록]

ui_text:
  title: 그룹 정보
  empty_message: ''
  error_message: 그룹 정보를 처리하지 못했습니다.
  toast: [그룹이 저장되었습니다., 그룹이 삭제되었습니다., 그룹을 처리하지 못했습니다.]
  confirm_message: [이 그룹을 삭제할까요?, 저장하지 않고 나갈까요?]

test_points:
  - 조직 내 그룹명 중복 검증 확인
  - 소속 차량 존재 시 삭제 제한 확인
```

### 차량-기사 연결 관리
```yaml
screen:
  id: SCR-LINK-001
  name: 차량-기사 연결 관리
  service: TrackCar Mobile
  domain: 조직운영
  route: /workspace/vehicle-driver-links
  menu_path: Workspace > 차량-기사 연결 관리
  roles: [OWNER]
  purpose: 차량과 담당 기사의 상시 연결 상태를 조회하고 연결/변경/해제한다.

layout:
  type: list
  platform: mobile
  navigation:
    header: 타이틀 + 필터
    body: 미연결 요약 + 연결 리스트
    footer: 없음

data_scope:
  primary_entity: vehicle_driver_link
  params: [organization_id, group_id, assigned_status, keyword, page, size]
  filters: [group_id, assigned_status]
  sort: [assigned_status asc, vehicle_no asc]
  pagination:
    enabled: true
    type: pagination

components:
  - id: cmp_unassigned_summary
    type: summary_card
    label: 미연결 차량 요약
    required: false
    readonly: true
    visible: true
    enabled: true
    options: [미연결 차량 수]
    data_binding: links.summary
    description: 우선 조치가 필요한 차량 수 표시
  - id: cmp_keyword
    type: search
    label: 차량번호/기사명 검색
    required: false
    readonly: false
    visible: true
    enabled: true
    options: debounce 500ms
    data_binding: query.keyword
    description: 차량 또는 기사 검색
  - id: cmp_filter
    type: bottom_sheet
    label: 조건 필터
    required: false
    readonly: false
    visible: true
    enabled: true
    options: [그룹, 연결상태]
    data_binding: query.*
    description: 연결 관리 필터
  - id: cmp_link_list
    type: list
    label: 연결 리스트
    required: false
    readonly: true
    visible: true
    enabled: true
    options: [차량, 그룹, 현재기사, 연결상태, 변경버튼]
    data_binding: links.items
    description: 차량별 연결 현황 리스트

actions:
  - trigger: 화면 진입
    target: 차량-기사 연결 관리
    behavior: 연결 현황 조회
    api_ref: GET /v1/mobile/vehicle-driver-links
    success_result: 리스트 표시
    error_result: 오류 상태 표시
  - trigger: 연결 저장
    target: 차량-기사 연결 관리
    behavior: 차량-기사 연결 생성 또는 변경
    api_ref: PUT /v1/mobile/vehicle-driver-links/{vehicleId}
    success_result: 목록 갱신 및 토스트 표시
    error_result: 오류 토스트 표시
  - trigger: 연결 해제
    target: 차량-기사 연결 관리
    behavior: 현재 기사 연결 해제
    api_ref: DELETE /v1/mobile/vehicle-driver-links/{vehicleId}
    success_result: 연결상태 미연결 갱신
    error_result: 오류 토스트 표시

states:
  initial: 스켈레톤 리스트 표시
  normal: 연결 현황 표시
  loading: 목록/저장 로딩 표시
  empty: 관리할 차량이 없습니다.
  error: 연결 현황을 불러오지 못했습니다.
  no_permission: 연결 관리 권한이 없습니다.
  conditional:
    - 미연결 차량이 있으면 요약 카드 강조
    - 연결된 기사 없는 경우 미연결 배지 표시

validation:
  driver_id:
    rule: 연결 저장 시 필수
    message: 연결할 기사를 선택해 주세요.

permissions:
  view: OWNER
  create: OWNER
  update: OWNER
  delete: OWNER
  field_control: []

api:
  method: GET/PUT/DELETE
  endpoint: /v1/mobile/vehicle-driver-links
  purpose: 차량-기사 상시 연결 조회 및 변경
  request:
    get:
      query:
        organization_id: string
        group_id: string|null
        assigned_status: string|null
        keyword: string|null
        page: number
        size: number
    put:
      body:
        vehicle_id: string
        driver_id: string
    delete:
      body:
        vehicle_id: string
  response:
    items: array
    summary:
      unassigned_vehicle_count: number

navigation_rules:
  entry_points: [워크스페이스 홈, 차량 상세 담당기사 변경]
  exits: [차량 상세, 워크스페이스 홈]

ui_text:
  title: 차량-기사 연결 관리
  empty_message: 관리할 차량이 없습니다.
  error_message: 연결 현황을 불러오지 못했습니다.
  toast: [차량과 기사가 연결되었습니다., 연결이 해제되었습니다., 연결을 변경하지 못했습니다.]
  confirm_message: [현재 기사 연결을 해제할까요?]

test_points:
  - 미연결 차량 강조 확인
  - 연결 저장/해제 후 즉시 목록 갱신 확인
  - 차량 상세와 연결 관리 간 데이터 일관성 확인
```

### 내 정보 / 설정
```yaml
screen:
  id: SCR-MY-001
  name: 내 정보 / 설정
  service: TrackCar Mobile
  domain: 계정
  route: /my
  menu_path: My
  roles: [OWNER, STAFF]
  purpose: 내 계정 정보, 알림 설정, 앱 정보, 조직 정보를 조회한다.

layout:
  type: settings
  platform: mobile
  navigation:
    header: 타이틀
    body: 프로필 + 조직정보 + 알림설정 + 앱정보
    footer: 하단 탭바

data_scope:
  primary_entity: me
  params: [session_user_id]
  filters: []
  sort: []
  pagination:
    enabled: false

components:
  - id: cmp_profile
    type: profile_card
    label: 내 계정
    required: false
    readonly: true
    visible: true
    enabled: true
    options: [이름, 이메일, 역할]
    data_binding: me.profile
    description: 사용자 기본 정보
  - id: cmp_org_info
    type: key_value
    label: 소속 조직
    required: false
    readonly: true
    visible: true
    enabled: true
    options: [조직명, 기본그룹]
    data_binding: me.organization
    description: 소속 조직 정보
  - id: fld_push_enabled
    type: toggle
    label: 푸시 알림 수신
    required: false
    readonly: false
    visible: true
    enabled: true
    options: [on, off]
    data_binding: form.push_enabled
    description: 앱 푸시 알림 수신 여부
  - id: fld_email_enabled
    type: toggle
    label: 이메일 알림 수신
    required: false
    readonly: false
    visible: true
    enabled: true
    options: [on, off]
    data_binding: form.email_enabled
    description: 이메일 알림 수신 여부

actions:
  - trigger: 화면 진입
    target: 내 정보 / 설정
    behavior: 내 정보와 설정 값 조회
    api_ref: GET /v1/mobile/me
    success_result: 정보 표시
    error_result: 오류 상태 표시
  - trigger: 알림 토글 변경
    target: 내 정보 / 설정
    behavior: 즉시 설정 저장
    api_ref: PATCH /v1/mobile/me/notification-settings
    success_result: 토스트 표시
    error_result: 원복 후 오류 토스트 표시

states:
  initial: 스켈레톤 설정 화면 표시
  normal: 정보 표시
  loading: 토글 저장 중 로딩 표시
  empty: 해당 없음
  error: 내 정보를 불러오지 못했습니다.
  no_permission: 해당 없음
  conditional:
    - OWNER만 Workspace 탭 노출
    - STAFF는 역할 표시를 STAFF로 고정

validation: {}

permissions:
  view: OWNER, STAFF
  create: none
  update: OWNER, STAFF
  delete: none
  field_control: []

api:
  method: GET/PATCH
  endpoint: /v1/mobile/me
  purpose: 내 정보 조회 및 알림 설정 변경
  request:
    patch_notification_settings:
      body:
        push_enabled: boolean
        email_enabled: boolean
  response:
    me: object

navigation_rules:
  entry_points: [하단 탭 My]
  exits: [로그인, 이용약관 페이지]

ui_text:
  title: 내 정보
  empty_message: ''
  error_message: 내 정보를 불러오지 못했습니다.
  toast: [설정이 저장되었습니다., 설정을 저장하지 못했습니다.]
  confirm_message: [로그아웃할까요?]

test_points:
  - 토글 변경 즉시 저장 및 실패 시 원복 확인
  - 역할별 표시값 확인
```

---

## 2. 변경 이력 (Changelog)

- **v1.0 (2026-03-25):**
  - 기존 화면설계서(trackcar_mobile_screen_spec_v1.md)를 리네이밍 및 문서 구조 개선
  - 문서 명명 규칙(TrackCar Mobile App *.md) 적용
  - 메타데이터(작성일, 버전, 상태) 추가
  - 공통 상태 코드 섹션 추가
  - 공통 UI 규칙 추가
  - Backend 문서 참조: eTAS 전송용 필드 보강
    - 차량 상세: `vehicle_type` (자동차유형코드) 필드 추가
    - 기사 상세: `driver_code` (운전자코드) 필드 추가
    - VIN 검증: 17자리 형식 명시
    - 차량번호 검증: `[별표 4]` 형식 안내

---

## 3. 참고 문서

### Backend 문서

| 문서명 | 버전 | 요약 |
|--------|------|------|
| TrackCar 플랫폼 아키텍처 상세 설계 | v2.4.0 | 전체 시스템 아키텍처, 데이터 흐름 |
| DTG 단말기 ↔ TrackCar IoT Core 연동 요구사항 | v1.6 | MQTT/mTLS 기반 DTG 연동 프로토콜 |
| TrackCar Ingest 검증 명세서 | v2.2 | DTG 데이터 수집, 검증, 저장 파이프라인 |
| TrackCar 외부 연계 전송 처리 명세서 | v1.1 | eTAS 전송 파이프라인, 배치 스케줄링 |

### 규정/기준 문서

| 문서명 | 출처 | 요약 |
|--------|------|------|
| 자동차운행기록 및 장치에 관한 관리지침 | 국토교통부 | DTG 설치 의무, 운행기록 관리 기준 |
| [별표 1] 운행기록장치 세부기준 | 관리지침 별표 | DTG 기술规格, 설치 기준 |
| [별표 2] 운행기록 파일 작성기준 | 관리지침 별표 | 운행기록 파일 배열순서 (운전자코드 포함) |
| [별표 3] 파일명 규격 | 관리지침 별표 | 파일명 생성 규칙 |
| [별표 4] 차량번호 형식 | 관리지침 별표 | 차량번호 형식 검증 기준 |
