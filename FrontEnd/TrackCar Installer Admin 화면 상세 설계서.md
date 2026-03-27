# TrackCar Installer Admin 화면 상세 설계서

- 작성일: 2026-03-25
- 버전: v1.0
- 상태: 초안

---

## 1. 문서 개요

### 1.1 목적

본 문서는 Installer Admin Web의 화면별 상세 설계를 정의한다. AI Agent가 바로 구현 가능한 수준의 기능 중심 화면 명세를 제공한다.

### 1.2 설계 기준

- Installer Admin은 설치/등록/매핑/검증/앱 활성화 확인을 담당한다.
- Mobile App은 실제 운행/관제/알림/사용자 설정을 담당한다.
- 설치 완료와 서비스 활성화 완료는 분리한다.
- owner / member / driver 계정 모델을 구분한다.
- Group은 Installer Admin에서 마스터 관리한다.

### 1.3 공통 상태 코드

- 차량 상태: REGISTERED, DEVICE_LINKED, DRIVER_LINKED, VERIFIED, ACTIVE, INACTIVE
- 운행 상태: UNKNOWN, STOPPED, IDLING, DRIVING, OFFLINE
- 설치 상태: NOT_INSTALLED, INSTALLED, VERIFIED, FAILED, REPLACED
- 알림 심각도: INFO, WARNING, CRITICAL
- 검증 상태: NOT_CHECKED, CHECKING, PASS, WARNING, FAIL
- 앱 활성화 상태: NOT_READY, INVITED, FIRST_LOGIN_DONE, LIVE, LOCKED

### 1.4 공통 UI 규칙

- 목록 화면은 기본 정렬, 필터, 페이지네이션을 지원한다.
- 실시간 상태가 포함된 화면은 자동 갱신 주기와 마지막 갱신 시각을 표시한다.
- 읽기 전용 필드는 `readonly: true`로 명시한다.
- 수정 불가 필드 제어는 `permissions.field_control`에 명시한다.
- API 명세는 구현 가정 기준이며, 추후 BFF 또는 Gateway 정책에 맞게 조정 가능하다.

---

## 2. 화면 상세 설계

### 2.1 대시보드
```yaml
screen:
  id: SCR-DASH-001
  name: 설치 운영 대시보드
  service: Installer Admin Web
  domain: Installation Monitoring
  route: /dashboard
  menu_path: Dashboard
  roles: [Installer, Operator, Admin]
  purpose: 설치 차량, DTG 연동, 검증, 앱 활성화 현황을 실시간으로 파악한다.
layout:
  type: dashboard
  platform: web-desktop
  navigation:
    header: global_header
    sidebar: main_sidebar
    tabs: []
    refresh_policy:
      auto_refresh: true
      interval_sec: 60
      show_last_refreshed_at: true
      manual_refresh_button: true
data_scope:
  primary_entity: dashboard_metrics
  params:
    - date_from
    - date_to
    - owner_type
    - group_id
    - device_model
    - verification_status
  filters:
    - id: date_range
      type: daterange
    - id: owner_type
      type: select
      values: [ALL, PERSONAL, BUSINESS]
    - id: group_id
      type: async_select
    - id: device_model
      type: async_select
    - id: verification_status
      type: multi_select
      values: [ALL, PASS, WARNING, FAIL, NOT_CHECKED]
  sort:
    default: metric_date.desc
    allowed: [metric_date.desc, metric_date.asc]
  pagination:
    enabled: false
components:
  - id: filter_date_range
    type: date_range_picker
    label: 조회 기간
    required: true
    readonly: false
    visible: true
    enabled: true
    options: {default: last_30_days}
    data_binding: query.date_range
    description: KPI와 차트 집계 기간을 제어한다.
  - id: kpi_installed_vehicle_count
    type: stat_card
    label: 총 설치 차량 수
    required: false
    readonly: true
    visible: true
    enabled: true
    options: {format: number}
    data_binding: metrics.installed_vehicle_count
    description: 설치 완료 차량 총 대수
  - id: kpi_online_device_count
    type: stat_card
    label: 활성 DTG 수
    required: false
    readonly: true
    visible: true
    enabled: true
    options: {format: number}
    data_binding: metrics.online_device_count
    description: 최종 수신 시각 기준 온라인 상태 장치 수
  - id: kpi_unlinked_device_count
    type: stat_card
    label: 미연결 DTG 수
    required: false
    readonly: true
    visible: true
    enabled: true
    options: {format: number}
    data_binding: metrics.unlinked_device_count
    description: 차량 미매핑 또는 미설치 장치 수
  - id: kpi_verification_pass_count
    type: stat_card
    label: 검증 PASS 수
    required: false
    readonly: true
    visible: true
    enabled: true
    options: {format: number}
    data_binding: metrics.verification_pass_count
    description: 최근 검증 결과 PASS 건수
  - id: chart_install_trend
    type: line_chart
    label: 설치 차량 추이
    required: false
    readonly: true
    visible: true
    enabled: true
    options: {x: date, y: installed_count}
    data_binding: charts.install_trend
    description: 일/주/월 단위 설치 완료 차량 추이
  - id: chart_device_status
    type: donut_chart
    label: DTG 연동 상태
    required: false
    readonly: true
    visible: true
    enabled: true
    options: {series: [ONLINE, OFFLINE, NEVER_CONNECTED, ERROR]}
    data_binding: charts.device_status
    description: 장치 연동 상태 분포
  - id: chart_vehicle_driver_mapping
    type: bar_chart
    label: 차량-기사 매핑 상태
    required: false
    readonly: true
    visible: true
    enabled: true
    options: {series: [MAPPED, VEHICLE_ONLY, DRIVER_ONLY, UNMAPPED]}
    data_binding: charts.vehicle_driver_mapping
    description: 차량과 기사의 연결 상태 분포
  - id: chart_activation_status
    type: funnel_chart
    label: 서비스 활성화 단계
    required: false
    readonly: true
    visible: true
    enabled: true
    options: {steps: [ACCOUNT_CREATED, DEVICE_REGISTERED, VEHICLE_MAPPED, DRIVER_MAPPED, APP_INVITED, FIRST_LOGIN_DONE, LIVE]}
    data_binding: charts.activation_funnel
    description: 설치 완료 대비 서비스 활성화 완료 전환 현황
  - id: table_recent_issue_vehicles
    type: data_table
    label: 최근 이상 차량
    required: false
    readonly: true
    visible: true
    enabled: true
    options:
      columns: [vehicle_no, device_serial_no, vehicle_status, installation_status, severity, last_received_at, location_summary]
      row_click_action: go_vehicle_detail
    data_binding: lists.recent_issue_vehicles
    description: 장애 또는 검증 실패 차량 목록
  - id: text_last_refreshed_at
    type: text
    label: 마지막 갱신 시각
    required: false
    readonly: true
    visible: true
    enabled: true
    options: {format: datetime}
    data_binding: meta.last_refreshed_at
    description: 대시보드 서버 응답 기준 최종 갱신 시각
actions:
  - trigger: on_load
    target: dashboard_page
    behavior: fetch_dashboard_metrics
    api_ref: GET /api/admin/dashboard
    success_result: KPI, 차트, 이상 차량 목록을 표시한다.
    error_result: error 상태와 재시도 버튼을 표시한다.
  - trigger: on_change_filter
    target: dashboard_page
    behavior: refetch_dashboard_metrics
    api_ref: GET /api/admin/dashboard
    success_result: 필터 기준으로 화면을 다시 집계한다.
    error_result: 기존 화면 유지 후 토스트를 표시한다.
  - trigger: on_click_row_recent_issue_vehicle
    target: table_recent_issue_vehicles
    behavior: navigate_vehicle_detail
    api_ref: null
    success_result: 차량 상세 화면으로 이동한다.
    error_result: 이동 불가 토스트를 표시한다.
states:
  initial: 기본 조회 기간은 최근 30일이며 첫 진입 시 자동 조회한다.
  normal: KPI, 차트, 목록이 모두 정상 노출된다.
  loading: 스켈레톤 카드와 차트 로더를 표시한다.
  empty: 해당 조건의 집계 데이터가 없습니다 메시지를 표시한다.
  error: 대시보드 데이터를 불러오지 못했습니다. 다시 시도해 주세요.
  no_permission: 대시보드 조회 권한이 없습니다.
  conditional:
    - condition: recent_issue_vehicles.length > 0
      result: 이상 차량 섹션을 강조 표시한다.
    - condition: metrics.online_device_count == 0
      result: DTG 수신 상태 확인 배너를 표시한다.
validation:
  date_range:
    rules:
      - 기간 시작일은 종료일보다 이후일 수 없다.
      - 조회 기간은 최대 365일까지 허용한다.
    messages:
      invalid_range: 조회 시작일과 종료일을 확인해 주세요.
      max_range: 조회 기간은 최대 365일까지 가능합니다.
permissions:
  view: [Installer, Operator, Admin]
  create: []
  update: []
  delete: []
  field_control:
    - field: group_id
      editable_roles: [Operator, Admin]
    - field: device_model
      editable_roles: [Installer, Operator, Admin]
api:
  - method: GET
    endpoint: /api/admin/dashboard
    purpose: 대시보드 KPI, 차트, 이상 차량 목록 조회
    request:
      query:
        date_from: string
        date_to: string
        owner_type: string
        group_id: string
        device_model: string
        verification_status: array
    response:
      metrics: object
      charts: object
      lists: object
      meta:
        last_refreshed_at: datetime
navigation_rules:
  entry_points:
    - 로그인 후 기본 랜딩 화면
    - 좌측 메뉴 Dashboard 클릭
  exits:
    - 차량 상세 화면
    - DTG 상세 화면
    - 검증 센터 화면
ui_text:
  title: 설치 운영 대시보드
  empty_message: 조회 조건에 해당하는 설치/연동 데이터가 없습니다.
  error_message: 대시보드 데이터를 불러오지 못했습니다.
  toast:
    refresh_success: 최신 현황으로 갱신했습니다.
    refresh_error: 갱신 중 오류가 발생했습니다.
  confirm_message: {}
test_points:
  - 최근 30일 기본 조회가 정상 적용되는가
  - 자동 갱신 시 마지막 갱신 시각이 갱신되는가
  - 이상 차량 행 클릭 시 상세 화면으로 이동하는가
  - 권한 없는 사용자가 접근 시 no_permission 상태가 표시되는가
```

### 2.2 Owner 목록/상세
```yaml
screen:
  id: SCR-OWN-001
  name: Owner 목록/상세
  service: Installer Admin Web
  domain: Owner Management
  route: /accounts/owners
  menu_path: Accounts > Owner Accounts
  roles: [Installer, Operator, Admin]
  purpose: 개인/법인 owner를 조회하고 상세 정보를 확인·수정한다.
layout:
  type: list_detail_split
  platform: web-desktop
  navigation:
    header: global_header
    sidebar: main_sidebar
    tabs: [list, detail]
data_scope:
  primary_entity: owner_account
  params: [keyword, owner_type, status, group_id, page, size, sort]
  filters:
    - id: keyword
      type: search
    - id: owner_type
      type: select
      values: [ALL, PERSONAL, BUSINESS]
    - id: status
      type: select
      values: [ALL, ACTIVE, INACTIVE, LOCKED]
    - id: group_id
      type: async_select
  sort:
    default: created_at.desc
    allowed: [created_at.desc, created_at.asc, name.asc, name.desc]
  pagination:
    enabled: true
    default_page_size: 20
    page_size_options: [20, 50, 100]
components:
  - id: search_keyword
    type: search_input
    label: 이름/회사명/휴대폰/사업자번호 검색
    required: false
    readonly: false
    visible: true
    enabled: true
    options: {debounce_ms: 300}
    data_binding: query.keyword
    description: owner 식별정보 기준 통합 검색
  - id: table_owner_list
    type: data_table
    label: Owner 목록
    required: false
    readonly: true
    visible: true
    enabled: true
    options:
      columns: [owner_id, owner_type, name_or_company_name, phone, business_no, transport_biz_no, default_group_name, vehicle_count, driver_count, app_activation_status, created_at]
      selectable: true
      row_click_action: load_detail
    data_binding: owners.items
    description: owner 계정 목록
  - id: panel_owner_detail
    type: detail_panel
    label: Owner 상세
    required: false
    readonly: false
    visible: true
    enabled: true
    options: {mode: read_edit_toggle}
    data_binding: owner_detail
    description: 선택된 owner 상세 정보
  - id: field_owner_id
    type: text
    label: Owner ID
    required: false
    readonly: true
    visible: true
    enabled: false
    options: {}
    data_binding: owner_detail.owner_id
    description: 시스템 식별자
  - id: field_owner_type
    type: badge
    label: Owner 유형
    required: true
    readonly: true
    visible: true
    enabled: false
    options: {values: [PERSONAL, BUSINESS]}
    data_binding: owner_detail.owner_type
    description: 개인/법인 유형
  - id: field_name_or_company
    type: text_input
    label: 이름/회사명
    required: true
    readonly: false
    visible: true
    enabled: true
    options: {max_length: 100}
    data_binding: owner_detail.name_or_company_name
    description: owner 표시명
  - id: field_phone
    type: phone_input
    label: 대표 휴대폰번호
    required: true
    readonly: false
    visible: true
    enabled: true
    options: {format: KR_MOBILE}
    data_binding: owner_detail.phone
    description: 로그인/초대 대상 번호
  - id: field_business_no
    type: business_no_input
    label: 사업자번호
    required: false
    readonly: false
    visible: true
    enabled: true
    options: {visible_when: owner_type == BUSINESS}
    data_binding: owner_detail.business_no
    description: 법인 owner용 사업자번호
  - id: field_transport_biz_no
    type: text_input
    label: 운송사업자등록번호
    required: false
    readonly: false
    visible: true
    enabled: true
    options: {visible_when: owner_type == BUSINESS, max_length: 20}
    data_binding: owner_detail.transport_biz_no
    description: eTAS 전송용 운송사업자등록번호 ([별표 2] 참조)
  - id: field_default_group_name
    type: text
    label: 기본 그룹
    required: false
    readonly: true
    visible: true
    enabled: false
    options: {}
    data_binding: owner_detail.default_group_name
    description: 자동 생성된 기본 그룹명
  - id: field_status
    type: select
    label: 상태
    required: true
    readonly: false
    visible: true
    enabled: true
    options: {values: [ACTIVE, INACTIVE, LOCKED]}
    data_binding: owner_detail.status
    description: owner 상태 제어
  - id: button_create_owner
    type: button
    label: Owner 등록
    required: false
    readonly: false
    visible: true
    enabled: true
    options: {variant: primary}
    data_binding: null
    description: Owner 등록 화면으로 이동
  - id: button_save_owner
    type: button
    label: 저장
    required: false
    readonly: false
    visible: true
    enabled: true
    options: {variant: primary}
    data_binding: null
    description: 상세 수정 저장
actions:
  - trigger: on_load
    target: owner_list_page
    behavior: fetch_owner_list
    api_ref: GET /api/admin/owners
    success_result: owner 목록을 표시한다.
    error_result: 목록 조회 오류 메시지를 표시한다.
  - trigger: on_click_row
    target: table_owner_list
    behavior: fetch_owner_detail
    api_ref: GET /api/admin/owners/{ownerId}
    success_result: 우측 상세 패널에 owner 정보를 표시한다.
    error_result: 상세 데이터를 불러오지 못했습니다.
  - trigger: on_click_create_owner
    target: button_create_owner
    behavior: navigate_create_screen
    api_ref: null
    success_result: Owner 등록 화면으로 이동한다.
    error_result: 이동 오류 토스트를 표시한다.
  - trigger: on_click_save_owner
    target: button_save_owner
    behavior: update_owner
    api_ref: PUT /api/admin/owners/{ownerId}
    success_result: 수정이 저장되었습니다 토스트를 표시하고 목록을 갱신한다.
    error_result: 저장 중 오류가 발생했습니다.
states:
  initial: 목록만 표시되고 첫 행 자동 선택은 하지 않는다.
  normal: 목록과 상세 패널이 동시에 표시된다.
  loading: 목록/상세 각 영역에 독립 로더를 표시한다.
  empty: 검색 조건에 맞는 Owner가 없습니다.
  error: Owner 정보를 불러오지 못했습니다.
  no_permission: Owner 조회 권한이 없습니다.
  conditional:
    - condition: owner_detail.owner_type == BUSINESS
      result: 사업자번호 필드를 표시한다.
    - condition: owner_detail.status == LOCKED
      result: 앱 초대 재발송 버튼을 비활성화한다.
validation:
  name_or_company_name:
    rules:
      - 필수 입력
      - 2자 이상 100자 이하
    messages:
      required: 이름 또는 회사명을 입력해 주세요.
      length: 이름 또는 회사명은 2자 이상 100자 이하로 입력해 주세요.
  phone:
    rules:
      - 필수 입력
      - 대한민국 휴대폰 형식
    messages:
      required: 대표 휴대폰번호를 입력해 주세요.
      format: 휴대폰번호 형식을 확인해 주세요.
  business_no:
    rules:
      - owner_type이 BUSINESS인 경우 필수
      - 사업자번호 형식 검증
    messages:
      required: 법인 Owner는 사업자번호가 필요합니다.
      format: 사업자번호 형식을 확인해 주세요.
permissions:
  view: [Installer, Operator, Admin]
  create: [Installer, Operator, Admin]
  update: [Operator, Admin]
  delete: []
  field_control:
    - field: owner_type
      editable_roles: []
    - field: status
      editable_roles: [Operator, Admin]
    - field: business_no
      editable_roles: [Operator, Admin]
api:
  - method: GET
    endpoint: /api/admin/owners
    purpose: Owner 목록 조회
    request:
      query:
        keyword: string
        owner_type: string
        status: string
        group_id: string
        page: number
        size: number
        sort: string
    response:
      items: array
      page_info: object
  - method: GET
    endpoint: /api/admin/owners/{ownerId}
    purpose: Owner 상세 조회
    request: {path: {ownerId: string}}
    response: {owner: object}
  - method: PUT
    endpoint: /api/admin/owners/{ownerId}
    purpose: Owner 정보 수정
    request:
      path: {ownerId: string}
      body: {name_or_company_name: string, phone: string, business_no: string, status: string}
    response: {owner: object}
navigation_rules:
  entry_points:
    - 메뉴 Accounts > Owner Accounts 클릭
    - 대시보드 KPI 또는 이상 차량 관련 owner 링크 클릭
  exits:
    - Owner 등록/수정 화면
    - Group 목록 화면
    - App Activation 화면
ui_text:
  title: Owner 계정 관리
  empty_message: 등록된 Owner가 없습니다.
  error_message: Owner 데이터를 불러오지 못했습니다.
  toast:
    save_success: Owner 정보가 저장되었습니다.
    save_error: Owner 저장 중 오류가 발생했습니다.
  confirm_message:
    change_status: Owner 상태를 변경하시겠습니까?
test_points:
  - PERSONAL/BUSINESS 유형에 따라 사업자번호 필드 표시가 달라지는가
  - 목록 검색/정렬/페이지네이션이 정상 동작하는가
  - Installer는 수정 불가, Operator/Admin만 수정 가능한가
```

### 2.3 Owner 등록/수정
```yaml
screen:
  id: SCR-OWN-002
  name: Owner 등록/수정
  service: Installer Admin Web
  domain: Owner Management
  route: /accounts/owners/new
  menu_path: Accounts > Owner Accounts > Owner 등록
  roles: [Installer, Operator, Admin]
  purpose: 신규 owner를 생성하고 기본 그룹 생성 정책과 앱 초대 방식을 지정한다.
layout:
  type: form
  platform: web-desktop
  navigation:
    header: global_header
    sidebar: main_sidebar
    tabs: [basic_info, group_policy, invite_policy]
data_scope:
  primary_entity: owner_account
  params: [mode, owner_id]
  filters: []
  sort: {default: null, allowed: []}
  pagination: {enabled: false}
components:
  - id: field_mode
    type: hidden
    label: 화면 모드
    required: true
    readonly: true
    visible: false
    enabled: false
    options: {values: [CREATE, EDIT]}
    data_binding: form.mode
    description: 등록 또는 수정 모드
  - id: field_owner_type
    type: radio_group
    label: Owner 유형
    required: true
    readonly: false
    visible: true
    enabled: true
    options: {values: [PERSONAL, BUSINESS]}
    data_binding: form.owner_type
    description: 개인/법인 선택
  - id: field_name_or_company
    type: text_input
    label: 이름/회사명
    required: true
    readonly: false
    visible: true
    enabled: true
    options: {max_length: 100}
    data_binding: form.name_or_company_name
    description: 계정 대표명
  - id: field_phone
    type: phone_input
    label: 대표 휴대폰번호
    required: true
    readonly: false
    visible: true
    enabled: true
    options: {format: KR_MOBILE}
    data_binding: form.phone
    description: SMS 초대 및 대표 연락처
  - id: field_email
    type: email_input
    label: 이메일
    required: false
    readonly: false
    visible: true
    enabled: true
    options: {max_length: 100}
    data_binding: form.email
    description: 선택 입력
  - id: field_business_no
    type: business_no_input
    label: 사업자번호
    required: false
    readonly: false
    visible: true
    enabled: true
    options: {visible_when: form.owner_type == BUSINESS}
    data_binding: form.business_no
    description: 법인 owner 식별자
  - id: field_transport_biz_no
    type: text_input
    label: 운송사업자등록번호
    required: false
    readonly: false
    visible: true
    enabled: true
    options: {visible_when: form.owner_type == BUSINESS, max_length: 20}
    data_binding: form.transport_biz_no
    description: eTAS 전송용 ([별표 2] 참조)
  - id: field_create_default_group
    type: checkbox
    label: 기본 그룹 자동 생성
    required: false
    readonly: false
    visible: true
    enabled: true
    options: {default: true}
    data_binding: form.create_default_group
    description: owner 생성과 함께 기본 그룹 생성 여부
  - id: field_default_group_name
    type: text_input
    label: 기본 그룹명
    required: false
    readonly: false
    visible: true
    enabled: true
    options: {placeholder: 기본그룹, visible_when: form.create_default_group == true}
    data_binding: form.default_group_name
    description: 자동 생성 그룹명
  - id: field_invite_method
    type: radio_group
    label: 앱 초대 방식
    required: true
    readonly: false
    visible: true
    enabled: true
    options: {values: [SMS_LINK, TEMP_PASSWORD, LATER]}
    data_binding: form.invite_method
    description: 앱 초대 또는 추후 처리 선택
  - id: button_check_duplicate
    type: button
    label: 중복 확인
    required: false
    readonly: false
    visible: true
    enabled: true
    options: {variant: secondary}
    data_binding: null
    description: 휴대폰번호/사업자번호 중복 확인
  - id: button_submit
    type: button
    label: 저장
    required: false
    readonly: false
    visible: true
    enabled: true
    options: {variant: primary}
    data_binding: null
    description: Owner 생성 또는 수정
  - id: button_cancel
    type: button
    label: 취소
    required: false
    readonly: false
    visible: true
    enabled: true
    options: {variant: tertiary}
    data_binding: null
    description: 이전 화면으로 이동
actions:
  - trigger: on_click_check_duplicate
    target: button_check_duplicate
    behavior: validate_owner_duplicate
    api_ref: POST /api/admin/owners/check-duplicate
    success_result: 사용 가능한 정보입니다 메시지를 표시한다.
    error_result: 중복된 휴대폰번호 또는 사업자번호입니다.
  - trigger: on_click_submit
    target: button_submit
    behavior: create_or_update_owner
    api_ref: POST /api/admin/owners or PUT /api/admin/owners/{ownerId}
    success_result: 저장 후 Owner 목록 또는 상세 화면으로 이동한다.
    error_result: 필드 오류 메시지와 저장 실패 토스트를 표시한다.
  - trigger: on_click_cancel
    target: button_cancel
    behavior: confirm_leave
    api_ref: null
    success_result: 미저장 변경이 없으면 이전 화면으로 이동한다.
    error_result: null
states:
  initial: mode가 CREATE이면 빈 폼으로 시작한다.
  normal: 필드 입력 및 중복 확인, 저장이 가능하다.
  loading: 저장 버튼 로딩 상태와 폼 비활성화를 적용한다.
  empty: 해당 없음
  error: Owner 저장 중 오류가 발생했습니다.
  no_permission: Owner 등록 권한이 없습니다.
  conditional:
    - condition: form.owner_type == BUSINESS
      result: 사업자번호를 필수로 전환한다.
    - condition: form.create_default_group == false
      result: 기본 그룹명 필드를 숨긴다.
validation:
  owner_type:
    rules: [필수 선택]
    messages:
      required: Owner 유형을 선택해 주세요.
  name_or_company_name:
    rules: [필수 입력, 2자 이상, 100자 이하]
    messages:
      required: 이름 또는 회사명을 입력해 주세요.
      length: 이름 또는 회사명은 2자 이상 100자 이하로 입력해 주세요.
  phone:
    rules: [필수 입력, 대한민국 휴대폰 형식, 중복 확인 완료]
    messages:
      required: 대표 휴대폰번호를 입력해 주세요.
      format: 휴대폰번호 형식을 확인해 주세요.
      duplicate_check: 휴대폰번호 중복 확인이 필요합니다.
  business_no:
    rules: [BUSINESS인 경우 필수, 형식 검증, 중복 확인 완료]
    messages:
      required: 법인 Owner는 사업자번호를 입력해 주세요.
      format: 사업자번호 형식을 확인해 주세요.
      duplicate_check: 사업자번호 중복 확인이 필요합니다.
  default_group_name:
    rules: [기본 그룹 자동 생성 시 1자 이상 50자 이하]
    messages:
      length: 기본 그룹명은 1자 이상 50자 이하로 입력해 주세요.
  invite_method:
    rules: [필수 선택]
    messages:
      required: 앱 초대 방식을 선택해 주세요.
permissions:
  view: [Installer, Operator, Admin]
  create: [Installer, Operator, Admin]
  update: [Operator, Admin]
  delete: []
  field_control:
    - field: owner_type
      editable_roles: [Installer, Operator, Admin]
    - field: business_no
      editable_roles: [Operator, Admin]
api:
  - method: POST
    endpoint: /api/admin/owners/check-duplicate
    purpose: 휴대폰번호/사업자번호 중복 확인
    request:
      body: {owner_type: string, phone: string, business_no: string}
    response: {duplicated: boolean, duplicated_fields: array}
  - method: POST
    endpoint: /api/admin/owners
    purpose: Owner 생성
    request:
      body:
        owner_type: string
        name_or_company_name: string
        phone: string
        email: string
        business_no: string
        create_default_group: boolean
        default_group_name: string
        invite_method: string
    response: {owner_id: string, default_group_id: string}
  - method: PUT
    endpoint: /api/admin/owners/{ownerId}
    purpose: Owner 수정
    request:
      path: {ownerId: string}
      body: {name_or_company_name: string, phone: string, email: string, business_no: string, invite_method: string}
    response: {owner_id: string}
navigation_rules:
  entry_points:
    - Owner 목록 화면의 Owner 등록 버튼
    - Owner 상세 화면의 수정 버튼
  exits:
    - Owner 목록 화면
    - Owner 상세 화면
    - Group 상세 화면
ui_text:
  title: Owner 등록
  empty_message: ''
  error_message: Owner 저장 중 오류가 발생했습니다.
  toast:
    duplicate_ok: 사용 가능한 정보입니다.
    save_success: Owner가 저장되었습니다.
    save_error: Owner 저장에 실패했습니다.
  confirm_message:
    leave_without_save: 저장하지 않은 내용이 있습니다. 화면을 나가시겠습니까?
test_points:
  - BUSINESS 선택 시 사업자번호가 필수로 바뀌는가
  - 중복 확인 없이 저장이 차단되는가
  - 기본 그룹 자동 생성 옵션이 정상 반영되는가
  - SMS_LINK/TEMP_PASSWORD/LATER 중 초대 방식이 저장되는가
```

### 2.4 Group 목록/등록
```yaml
screen:
  id: SCR-GRP-001
  name: Group 목록/등록
  service: Installer Admin Web
  domain: Group Management
  route: /groups
  menu_path: Groups
  roles: [Installer, Operator, Admin]
  purpose: owner 산하 그룹을 생성하고 차량/기사/멤버 소속 기준을 관리한다.
layout:
  type: list_form_split
  platform: web-desktop
  navigation:
    header: global_header
    sidebar: main_sidebar
    tabs: [list, form]
data_scope:
  primary_entity: group
  params: [owner_id, keyword, status, page, size, sort]
  filters:
    - id: owner_id
      type: async_select
    - id: keyword
      type: search
    - id: status
      type: select
      values: [ALL, ACTIVE, INACTIVE]
  sort:
    default: created_at.desc
    allowed: [created_at.desc, created_at.asc, group_name.asc, group_name.desc]
  pagination:
    enabled: true
    default_page_size: 20
components:
  - id: select_owner
    type: async_select
    label: Owner 선택
    required: false
    readonly: false
    visible: true
    enabled: true
    options: {entity: owner}
    data_binding: query.owner_id
    description: owner 기준 그룹 조회
  - id: table_group_list
    type: data_table
    label: 그룹 목록
    required: false
    readonly: true
    visible: true
    enabled: true
    options:
      columns: [group_id, owner_name, group_name, member_count, vehicle_count, driver_count, status, created_at]
      row_click_action: populate_form
    data_binding: groups.items
    description: 그룹 목록
  - id: field_group_name
    type: text_input
    label: 그룹명
    required: true
    readonly: false
    visible: true
    enabled: true
    options: {max_length: 50}
    data_binding: form.group_name
    description: 그룹 표시명
  - id: field_group_status
    type: select
    label: 상태
    required: true
    readonly: false
    visible: true
    enabled: true
    options: {values: [ACTIVE, INACTIVE]}
    data_binding: form.status
    description: 그룹 상태
  - id: field_is_default_group
    type: checkbox
    label: 기본 그룹
    required: false
    readonly: false
    visible: true
    enabled: true
    options: {single_default_per_owner: true}
    data_binding: form.is_default_group
    description: owner별 기본 그룹 지정
  - id: button_save_group
    type: button
    label: 그룹 저장
    required: false
    readonly: false
    visible: true
    enabled: true
    options: {variant: primary}
    data_binding: null
    description: 그룹 생성/수정 저장
actions:
  - trigger: on_load
    target: group_page
    behavior: fetch_group_list
    api_ref: GET /api/admin/groups
    success_result: 그룹 목록 표시
    error_result: 그룹 목록 조회 실패
  - trigger: on_click_row
    target: table_group_list
    behavior: load_group_detail_to_form
    api_ref: GET /api/admin/groups/{groupId}
    success_result: 우측 폼에 그룹 상세를 채운다.
    error_result: 그룹 상세 조회 실패
  - trigger: on_click_save_group
    target: button_save_group
    behavior: create_or_update_group
    api_ref: POST /api/admin/groups or PUT /api/admin/groups/{groupId}
    success_result: 저장 후 목록 갱신 및 성공 토스트 표시
    error_result: 저장 실패 메시지 표시
states:
  initial: owner 미선택 시 전체 그룹을 조회한다.
  normal: 목록과 폼이 같이 표시된다.
  loading: 목록/폼 독립 로딩 표시
  empty: 등록된 그룹이 없습니다.
  error: 그룹 정보를 불러오지 못했습니다.
  no_permission: 그룹 관리 권한이 없습니다.
  conditional:
    - condition: form.is_default_group == true
      result: 같은 owner의 기존 기본 그룹은 자동 해제 예정 안내를 표시한다.
validation:
  owner_id:
    rules: [필수 선택]
    messages:
      required: Owner를 선택해 주세요.
  group_name:
    rules: [필수 입력, 1자 이상 50자 이하, owner 내 중복 불가]
    messages:
      required: 그룹명을 입력해 주세요.
      length: 그룹명은 1자 이상 50자 이하로 입력해 주세요.
      duplicate: 동일 Owner 내 같은 그룹명을 사용할 수 없습니다.
permissions:
  view: [Installer, Operator, Admin]
  create: [Installer, Operator, Admin]
  update: [Operator, Admin]
  delete: []
  field_control:
    - field: is_default_group
      editable_roles: [Operator, Admin]
api:
  - method: GET
    endpoint: /api/admin/groups
    purpose: 그룹 목록 조회
    request:
      query: {owner_id: string, keyword: string, status: string, page: number, size: number, sort: string}
    response: {items: array, page_info: object}
  - method: POST
    endpoint: /api/admin/groups
    purpose: 그룹 생성
    request:
      body: {owner_id: string, group_name: string, status: string, is_default_group: boolean}
    response: {group_id: string}
  - method: PUT
    endpoint: /api/admin/groups/{groupId}
    purpose: 그룹 수정
    request:
      path: {groupId: string}
      body: {group_name: string, status: string, is_default_group: boolean}
    response: {group_id: string}
navigation_rules:
  entry_points:
    - 좌측 메뉴 Groups 클릭
    - Owner 상세 화면의 그룹 보기 링크 클릭
  exits:
    - Owner 목록/상세
    - Vehicle 목록
    - Driver 목록
ui_text:
  title: 그룹 관리
  empty_message: 등록된 그룹이 없습니다.
  error_message: 그룹 데이터를 불러오지 못했습니다.
  toast:
    save_success: 그룹이 저장되었습니다.
    save_error: 그룹 저장 중 오류가 발생했습니다.
  confirm_message:
    overwrite_default_group: 기존 기본 그룹을 해제하고 새 기본 그룹으로 지정하시겠습니까?
test_points:
  - owner별 기본 그룹이 1개만 유지되는가
  - Installer는 등록 가능하나 기본 그룹 변경은 제한되는가
  - 그룹 저장 후 목록과 카운트가 갱신되는가
```

### 2.5 DTG 목록/상세
```yaml
screen:
  id: SCR-DTG-001
  name: DTG 목록/상세
  service: Installer Admin Web
  domain: Device Management
  route: /devices
  menu_path: DTG Devices
  roles: [Installer, Operator, Admin]
  purpose: DTG 기초정보, 설치 상태, 최종 수신 시각, 연결 차량 상태를 조회한다.
layout:
  type: list_detail_split
  platform: web-desktop
  navigation:
    header: global_header
    sidebar: main_sidebar
    tabs: [list, detail, history]
data_scope:
  primary_entity: dtg_device
  params: [keyword, owner_id, status, installation_status, page, size, sort]
  filters:
    - id: keyword
      type: search
    - id: owner_id
      type: async_select
    - id: status
      type: multi_select
      values: [PENDING_ACTIVATION, ACTIVE, REGISTERED, MAPPED, ONLINE, OFFLINE, ERROR]
    - id: installation_status
      type: select
      values: [ALL, NOT_INSTALLED, INSTALLED, VERIFIED, FAILED, REPLACED]
  sort:
    default: created_at.desc
    allowed: [created_at.desc, last_received_at.desc, serial_no.asc]
  pagination:
    enabled: true
    default_page_size: 20
components:
  - id: table_device_list
    type: data_table
    label: DTG 목록
    required: false
    readonly: true
    visible: true
    enabled: true
    options:
      columns: [device_id, serial_no, approval_no, product_serial_no, model_name, device_status, installation_status, vehicle_no, last_received_at, location_summary]
      row_click_action: load_detail
    data_binding: devices.items
    description: DTG 목록 및 실시간 상태
  - id: field_serial_no
    type: text
    label: 시리얼번호
    required: true
    readonly: true
    visible: true
    enabled: false
    options: {}
    data_binding: device_detail.serial_no
    description: 장치 식별값
  - id: field_approval_no
    type: text
    label: 형식승인번호
    required: true
    readonly: true
    visible: true
    enabled: false
    options: {}
    data_binding: device_detail.approval_no
    description: Admin 전용 기기 식별값
  - id: field_product_serial_no
    type: text
    label: 제품일련번호
    required: true
    readonly: true
    visible: true
    enabled: false
    options: {}
    data_binding: device_detail.product_serial_no
    description: 제조사 제품 일련번호
  - id: field_device_status
    type: status_badge
    label: 장치 상태
    required: false
    readonly: true
    visible: true
    enabled: false
    options: {values: [PENDING_ACTIVATION, ACTIVE, REGISTERED, MAPPED, ONLINE, OFFLINE, ERROR]}
    data_binding: device_detail.device_status
    description: DTG 활성화 상태 (PENDING_ACTIVATION: 개통 대기, ACTIVE: 활성화 완료, REGISTERED: 등록됨, MAPPED: 차량 연결됨, ONLINE/OFFLINE: 실시간 수신 상태)
  - id: field_installation_status
    type: status_badge
    label: 설치 상태
    required: false
    readonly: false
    visible: true
    enabled: true
    options: {values: [NOT_INSTALLED, INSTALLED, VERIFIED, FAILED, REPLACED]}
    data_binding: device_detail.installation_status
    description: 물리 설치 및 검증 상태
  - id: field_last_received_at
    type: text
    label: 최종 수신 시각
    required: false
    readonly: true
    visible: true
    enabled: false
    options: {format: datetime}
    data_binding: device_detail.last_received_at
    description: 서버 기준 마지막 메시지 수신 시각
  - id: field_location_summary
    type: text
    label: 최근 위치 정보
    required: false
    readonly: true
    visible: true
    enabled: false
    options: {}
    data_binding: device_detail.location_summary
    description: 최근 GPS 요약 정보
  - id: button_register_device
    type: button
    label: DTG 등록
    required: false
    readonly: false
    visible: true
    enabled: true
    options: {variant: primary}
    data_binding: null
    description: DTG 등록 화면 이동
  - id: button_reverify_device
    type: button
    label: 연동 재검증
    required: false
    readonly: false
    visible: true
    enabled: true
    options: {variant: secondary}
    data_binding: null
    description: 선택 장치 기준 연동 재검증 실행
actions:
  - trigger: on_load
    target: device_page
    behavior: fetch_device_list
    api_ref: GET /api/admin/devices
    success_result: DTG 목록 표시
    error_result: DTG 목록 조회 실패
  - trigger: on_click_row
    target: table_device_list
    behavior: fetch_device_detail
    api_ref: GET /api/admin/devices/{deviceId}
    success_result: DTG 상세 표시
    error_result: DTG 상세 조회 실패
  - trigger: on_click_reverify_device
    target: button_reverify_device
    behavior: run_device_reverification
    api_ref: POST /api/admin/verifications/run
    success_result: 재검증 요청이 접수되었습니다 토스트 표시
    error_result: 재검증 요청 실패 토스트 표시
states:
  initial: 최근 생성 순 목록을 표시한다.
  normal: 목록/상세/이력을 조회한다.
  loading: 목록 또는 상세 별도 로더 표시
  empty: 등록된 DTG가 없습니다.
  error: DTG 데이터를 불러오지 못했습니다.
  no_permission: DTG 조회 권한이 없습니다.
  conditional:
    - condition: device_detail.device_status == ERROR
      result: 상단에 CRITICAL 배너와 최근 오류 사유를 표시한다.
    - condition: device_detail.last_received_at == null
      result: 최초 연결 대기 문구를 표시한다.
validation:
  keyword:
    rules: [최대 100자]
    messages:
      length: 검색어는 100자 이하로 입력해 주세요.
permissions:
  view: [Installer, Operator, Admin]
  create: [Installer, Operator, Admin]
  update: [Operator, Admin]
  delete: []
  field_control:
    - field: approval_no
      editable_roles: [Admin]
    - field: product_serial_no
      editable_roles: [Admin]
    - field: installation_status
      editable_roles: [Operator, Admin]
api:
  - method: GET
    endpoint: /api/admin/devices
    purpose: DTG 목록 조회
    request:
      query: {keyword: string, owner_id: string, status: array, installation_status: string, page: number, size: number, sort: string}
    response: {items: array, page_info: object}
  - method: GET
    endpoint: /api/admin/devices/{deviceId}
    purpose: DTG 상세 조회
    request: {path: {deviceId: string}}
    response: {device: object, status_history: array}
  - method: POST
    endpoint: /api/admin/verifications/run
    purpose: 장치 기준 재검증 실행
    request:
      body: {target_type: DEVICE, target_id: string}
    response: {verification_request_id: string}
navigation_rules:
  entry_points:
    - 메뉴 DTG Devices 클릭
    - 대시보드 이상 장치 클릭
  exits:
    - DTG 등록 화면
    - 차량 상세 화면
    - 검증 센터
ui_text:
  title: DTG 기기 관리
  empty_message: 등록된 DTG가 없습니다.
  error_message: DTG 데이터를 불러오지 못했습니다.
  toast:
    reverification_requested: 재검증을 요청했습니다.
    reverification_failed: 재검증 요청에 실패했습니다.
  confirm_message: {}
test_points:
  - 최종 수신 시각이 없는 장치가 NEVER_CONNECTED 성격으로 노출되는가
  - 설치 상태 변경 권한이 Operator/Admin에만 있는가
  - 재검증 요청 후 토스트와 이력 반영이 정상인가
```

### 2.6 DTG 등록/수정
```yaml
screen:
  id: SCR-DTG-002
  name: DTG 등록/수정
  service: Installer Admin Web
  domain: Device Management
  route: /devices/new
  menu_path: DTG Devices > DTG 등록
  roles: [Installer, Operator, Admin]
  purpose: DTG 기기의 표준 식별정보와 설치 부가정보를 등록한다.
layout:
  type: form
  platform: web-desktop
  navigation:
    header: global_header
    sidebar: main_sidebar
    tabs: [basic_info, install_info]
data_scope:
  primary_entity: dtg_device
  params: [mode, device_id]
  filters: []
  sort: {default: null, allowed: []}
  pagination: {enabled: false}
components:
  - id: field_serial_no
    type: text_input
    label: 시리얼번호
    required: true
    readonly: false
    visible: true
    enabled: true
    options: {max_length: 50}
    data_binding: form.serial_no
    description: DTG 논리 시리얼번호
  - id: field_approval_no
    type: text_input
    label: 형식승인번호
    required: true
    readonly: false
    visible: true
    enabled: true
    options: {max_length: 50}
    data_binding: form.approval_no
    description: 표준사양 기반 식별값
  - id: field_product_serial_no
    type: text_input
    label: 제품일련번호
    required: true
    readonly: false
    visible: true
    enabled: true
    options: {max_length: 50}
    data_binding: form.product_serial_no
    description: 제조 제품 일련번호
  - id: field_model_name
    type: text_input
    label: 모델명
    required: true
    readonly: false
    visible: true
    enabled: true
    options: {max_length: 100}
    data_binding: form.model_name
    description: 장치 모델명
  - id: field_firmware_version
    type: text_input
    label: 펌웨어 버전
    required: false
    readonly: false
    visible: true
    enabled: true
    options: {max_length: 30}
    data_binding: form.firmware_version
    description: 장치 펌웨어 버전
  - id: field_line_number
    type: text_input
    label: 통신 회선 번호
    required: false
    readonly: false
    visible: true
    enabled: true
    options: {max_length: 20}
    data_binding: form.line_number
    description: SIM/회선 관리용 번호
  - id: field_installation_status
    type: select
    label: 설치 상태
    required: true
    readonly: false
    visible: true
    enabled: true
    options: {values: [NOT_INSTALLED, INSTALLED, VERIFIED, FAILED, REPLACED]}
    data_binding: form.installation_status
    description: 물리 설치 상태
  - id: field_install_note
    type: textarea
    label: 설치 메모
    required: false
    readonly: false
    visible: true
    enabled: true
    options: {max_length: 1000}
    data_binding: form.install_note
    description: 현장 특이사항 기록
  - id: button_check_duplicate
    type: button
    label: 시리얼 중복 확인
    required: false
    readonly: false
    visible: true
    enabled: true
    options: {variant: secondary}
    data_binding: null
    description: 시리얼/제품일련번호 중복 확인
  - id: button_submit
    type: button
    label: 저장
    required: false
    readonly: false
    visible: true
    enabled: true
    options: {variant: primary}
    data_binding: null
    description: DTG 저장
actions:
  - trigger: on_click_check_duplicate
    target: button_check_duplicate
    behavior: check_device_duplicate
    api_ref: POST /api/admin/devices/check-duplicate
    success_result: 사용 가능한 시리얼입니다.
    error_result: 이미 등록된 시리얼번호 또는 제품일련번호입니다.
  - trigger: on_click_submit
    target: button_submit
    behavior: create_or_update_device
    api_ref: POST /api/admin/devices or PUT /api/admin/devices/{deviceId}
    success_result: DTG 저장 후 상세 화면으로 이동한다.
    error_result: 저장 실패 시 필드 오류를 표시한다.
states:
  initial: 등록 모드는 빈 폼으로 시작한다.
  normal: 필드 입력, 중복 확인, 저장 가능
  loading: 저장 중 전체 폼 비활성화
  empty: 해당 없음
  error: DTG 저장 중 오류가 발생했습니다.
  no_permission: DTG 등록 권한이 없습니다.
  conditional:
    - condition: form.installation_status == FAILED
      result: 설치 메모 필드를 강조 표시한다.
validation:
  serial_no:
    rules: [필수 입력, 3자 이상 50자 이하, 중복 확인 완료]
    messages:
      required: 시리얼번호를 입력해 주세요.
      length: 시리얼번호는 3자 이상 50자 이하로 입력해 주세요.
      duplicate_check: 시리얼 중복 확인이 필요합니다.
  approval_no:
    rules: [필수 입력, 3자 이상 50자 이하]
    messages:
      required: 형식승인번호를 입력해 주세요.
      length: 형식승인번호를 확인해 주세요.
  product_serial_no:
    rules: [필수 입력, 3자 이상 50자 이하, 중복 확인 완료]
    messages:
      required: 제품일련번호를 입력해 주세요.
      duplicate_check: 제품일련번호 중복 확인이 필요합니다.
  model_name:
    rules: [필수 입력, 2자 이상 100자 이하]
    messages:
      required: 모델명을 입력해 주세요.
permissions:
  view: [Installer, Operator, Admin]
  create: [Installer, Operator, Admin]
  update: [Operator, Admin]
  delete: []
  field_control:
    - field: approval_no
      editable_roles: [Admin]
    - field: product_serial_no
      editable_roles: [Admin]
api:
  - method: POST
    endpoint: /api/admin/devices/check-duplicate
    purpose: 시리얼/제품일련번호 중복 확인
    request:
      body: {serial_no: string, product_serial_no: string}
    response: {duplicated: boolean, duplicated_fields: array}
  - method: POST
    endpoint: /api/admin/devices
    purpose: DTG 생성
    request:
      body:
        serial_no: string
        approval_no: string
        product_serial_no: string
        model_name: string
        firmware_version: string
        line_number: string
        installation_status: string
        install_note: string
    response: {device_id: string}
  - method: PUT
    endpoint: /api/admin/devices/{deviceId}
    purpose: DTG 수정
    request:
      path: {deviceId: string}
      body:
        model_name: string
        firmware_version: string
        line_number: string
        installation_status: string
        install_note: string
    response: {device_id: string}
navigation_rules:
  entry_points:
    - DTG 목록 화면의 DTG 등록 버튼
    - DTG 상세 화면의 수정 버튼
  exits:
    - DTG 목록/상세 화면
    - Quick Mapping 화면
ui_text:
  title: DTG 등록
  empty_message: ''
  error_message: DTG 저장 중 오류가 발생했습니다.
  toast:
    save_success: DTG 정보가 저장되었습니다.
    save_error: DTG 저장에 실패했습니다.
  confirm_message:
    leave_without_save: 저장하지 않은 내용이 있습니다. 화면을 나가시겠습니까?
test_points:
  - 시리얼/제품일련번호 중복 확인 없이 저장이 차단되는가
  - Admin만 형식승인번호/제품일련번호 수정 가능한가
  - 설치 실패 상태에서 메모가 강조되는가
```

### 2.7 차량 목록/등록상세
```yaml
screen:
  id: SCR-VEH-001
  name: 차량 목록/등록상세
  service: Installer Admin Web
  domain: Vehicle Management
  route: /vehicles
  menu_path: Vehicles
  roles: [Installer, Operator, Admin]
  purpose: 차량 기준정보와 DTG/기사 연결 상태, 최종 수신 상태를 관리한다.
layout:
  type: list_detail_split
  platform: web-desktop
  navigation:
    header: global_header
    sidebar: main_sidebar
    tabs: [list, detail, receive_status]
data_scope:
  primary_entity: vehicle
  params: [keyword, owner_id, group_id, vehicle_status, running_status, page, size, sort]
  filters:
    - id: keyword
      type: search
    - id: owner_id
      type: async_select
    - id: group_id
      type: async_select
    - id: vehicle_status
      type: multi_select
      values: [REGISTERED, DEVICE_LINKED, DRIVER_LINKED, VERIFIED, ACTIVE, INACTIVE]
    - id: running_status
      type: multi_select
      values: [UNKNOWN, STOPPED, IDLING, DRIVING, OFFLINE]
  sort:
    default: created_at.desc
    allowed: [created_at.desc, vehicle_no.asc, last_received_at.desc]
  pagination:
    enabled: true
    default_page_size: 20
components:
  - id: table_vehicle_list
    type: data_table
    label: 차량 목록
    required: false
    readonly: true
    visible: true
    enabled: true
    options:
      columns: [vehicle_id, vehicle_no, vehicle_type, vin, owner_name, group_name, vehicle_status, running_status, installation_status, device_serial_no, primary_driver_name, last_received_at, location_summary]
      row_click_action: load_detail
    data_binding: vehicles.items
    description: 차량 목록 및 상태
  - id: field_vehicle_no
    type: text_input
    label: 차량번호
    required: true
    readonly: false
    visible: true
    enabled: true
    options: {max_length: 20}
    data_binding: vehicle_detail.vehicle_no
    description: 차량 등록번호
  - id: field_vin
    type: text_input
    label: 차대번호(VIN)
    required: true
    readonly: false
    visible: true
    enabled: true
    options: {max_length: 17, pattern: '^[A-HJ-NPR-Z0-9]{17}$'}
    data_binding: vehicle_detail.vin
    description: 차량 식별 번호 (17자리 영문/숫자, I, O, Q 제외)
  - id: field_vehicle_type
    type: select
    label: 자동차유형코드
    required: false
    readonly: false
    visible: true
    enabled: true
    options: {values: [TRUCK, BUS, TAXI, SPECIAL, ETC], description: 'eTAS 전송용 ([별표 2] 참조)'}
    data_binding: vehicle_detail.vehicle_type
    description: eTAS 전송용 자동차유형코드 ([별표 2] 참조)
  - id: field_owner_id
    type: async_select
    label: Owner
    required: true
    readonly: false
    visible: true
    enabled: true
    options: {entity: owner}
    data_binding: vehicle_detail.owner_id
    description: 상위 owner 연결
  - id: field_group_id
    type: async_select
    label: Group
    required: true
    readonly: false
    visible: true
    enabled: true
    options: {entity: group, depends_on: owner_id}
    data_binding: vehicle_detail.group_id
    description: 소속 그룹
  - id: field_vehicle_status
    type: status_badge
    label: 차량 상태
    required: false
    readonly: true
    visible: true
    enabled: false
    options: {values: [REGISTERED, DEVICE_LINKED, DRIVER_LINKED, VERIFIED, ACTIVE, INACTIVE]}
    data_binding: vehicle_detail.vehicle_status
    description: 등록/매핑/활성 상태
  - id: field_running_status
    type: status_badge
    label: 운행 상태
    required: false
    readonly: true
    visible: true
    enabled: false
    options: {values: [UNKNOWN, STOPPED, IDLING, DRIVING, OFFLINE]}
    data_binding: vehicle_detail.running_status
    description: 최근 수신 데이터 기반 운행 상태
  - id: field_installation_status
    type: status_badge
    label: 설치 상태
    required: false
    readonly: true
    visible: true
    enabled: false
    options: {values: [NOT_INSTALLED, INSTALLED, VERIFIED, FAILED, REPLACED]}
    data_binding: vehicle_detail.installation_status
    description: 연결된 DTG 설치 상태 요약
  - id: field_last_received_at
    type: text
    label: 최종 수신 시각
    required: false
    readonly: true
    visible: true
    enabled: false
    options: {format: datetime}
    data_binding: vehicle_detail.last_received_at
    description: 실시간 수신 마지막 시각
  - id: field_location_summary
    type: text
    label: 최근 위치 정보
    required: false
    readonly: true
    visible: true
    enabled: false
    options: {}
    data_binding: vehicle_detail.location_summary
    description: 위도/경도 또는 주소 요약
  - id: button_save_vehicle
    type: button
    label: 저장
    required: false
    readonly: false
    visible: true
    enabled: true
    options: {variant: primary}
    data_binding: null
    description: 차량 정보 저장
  - id: button_open_mapping
    type: button
    label: 매핑 관리
    required: false
    readonly: false
    visible: true
    enabled: true
    options: {variant: secondary}
    data_binding: null
    description: 선택 차량 기준 매핑 화면 이동
actions:
  - trigger: on_load
    target: vehicle_page
    behavior: fetch_vehicle_list
    api_ref: GET /api/admin/vehicles
    success_result: 차량 목록 표시
    error_result: 차량 목록 조회 실패
  - trigger: on_click_row
    target: table_vehicle_list
    behavior: fetch_vehicle_detail
    api_ref: GET /api/admin/vehicles/{vehicleId}
    success_result: 차량 상세 표시
    error_result: 차량 상세 조회 실패
  - trigger: on_click_save_vehicle
    target: button_save_vehicle
    behavior: create_or_update_vehicle
    api_ref: POST /api/admin/vehicles or PUT /api/admin/vehicles/{vehicleId}
    success_result: 차량 정보 저장 후 목록 갱신
    error_result: 차량 저장 실패
states:
  initial: 목록 중심 화면으로 시작한다.
  normal: 차량 상태, 운행 상태, 최종 수신 시각을 표시한다.
  loading: 목록과 상세 독립 로더 표시
  empty: 등록된 차량이 없습니다.
  error: 차량 데이터를 불러오지 못했습니다.
  no_permission: 차량 조회 권한이 없습니다.
  conditional:
    - condition: vehicle_detail.running_status == OFFLINE
      result: 최종 수신 지연 배너를 표시한다.
    - condition: vehicle_detail.location_summary == null
      result: 최근 위치 정보 없음 문구를 표시한다.
validation:
  vehicle_no:
    rules: [필수 입력, 2자 이상 20자 이하, [별표 4] 차량번호 형식 검증, owner 내 중복 불가]
    messages:
      required: 차량번호를 입력해 주세요.
      format: 차량번호 형식을 확인해 주세요. ([별표 4] 참조)
      duplicate: 동일 Owner 내 같은 차량번호를 사용할 수 없습니다.
  vin:
    rules: [필수 입력, 17자리 영문/숫자]
    messages:
      required: 차대번호를 입력해 주세요.
      format: 차대번호 형식을 확인해 주세요.
  owner_id:
    rules: [필수 선택]
    messages:
      required: Owner를 선택해 주세요.
  group_id:
    rules: [필수 선택]
    messages:
      required: Group을 선택해 주세요.
permissions:
  view: [Installer, Operator, Admin]
  create: [Installer, Operator, Admin]
  update: [Operator, Admin]
  delete: []
  field_control:
    - field: vehicle_status
      editable_roles: []
    - field: running_status
      editable_roles: []
api:
  - method: GET
    endpoint: /api/admin/vehicles
    purpose: 차량 목록 조회
    request:
      query: {keyword: string, owner_id: string, group_id: string, vehicle_status: array, running_status: array, page: number, size: number, sort: string}
    response: {items: array, page_info: object}
  - method: GET
    endpoint: /api/admin/vehicles/{vehicleId}
    purpose: 차량 상세 조회
    request: {path: {vehicleId: string}}
    response: {vehicle: object}
  - method: POST
    endpoint: /api/admin/vehicles
    purpose: 차량 생성
    request:
      body: {vehicle_no: string, vin: string, owner_id: string, group_id: string, vehicle_type: string}
    response: {vehicle_id: string}
  - method: PUT
    endpoint: /api/admin/vehicles/{vehicleId}
    purpose: 차량 수정
    request:
      path: {vehicleId: string}
      body: {vehicle_no: string, vin: string, group_id: string, vehicle_type: string}
    response: {vehicle_id: string}
navigation_rules:
  entry_points:
    - 메뉴 Vehicles 클릭
    - 대시보드 이상 차량 클릭
  exits:
    - Quick Mapping 화면
    - 검증 센터
    - App Activation 화면
ui_text:
  title: 차량 관리
  empty_message: 등록된 차량이 없습니다.
  error_message: 차량 데이터를 불러오지 못했습니다.
  toast:
    save_success: 차량 정보가 저장되었습니다.
    save_error: 차량 저장에 실패했습니다.
  confirm_message: {}
test_points:
  - 차량 상태/운행 상태/설치 상태가 구분되어 표시되는가
  - 최종 수신 시각 및 위치 정보가 읽기 전용으로 노출되는가
  - Offline 차량 배너가 조건부로 표시되는가
```

### 2.8 기사 목록/등록상세
```yaml
screen:
  id: SCR-DRV-001
  name: 기사 목록/등록상세
  service: Installer Admin Web
  domain: Driver Management
  route: /drivers
  menu_path: Drivers
  roles: [Installer, Operator, Admin]
  purpose: 기사 기준정보와 계정 연결 여부, 기본 차량 연결 상태를 관리한다.
layout:
  type: list_detail_split
  platform: web-desktop
  navigation:
    header: global_header
    sidebar: main_sidebar
    tabs: [list, detail, account_link]
data_scope:
  primary_entity: driver
  params: [keyword, owner_id, group_id, account_linked, status, page, size, sort]
  filters:
    - id: keyword
      type: search
    - id: owner_id
      type: async_select
    - id: group_id
      type: async_select
    - id: account_linked
      type: select
      values: [ALL, YES, NO]
    - id: status
      type: select
      values: [ALL, ACTIVE, INACTIVE]
  sort:
    default: created_at.desc
    allowed: [created_at.desc, driver_name.asc, updated_at.desc]
  pagination:
    enabled: true
    default_page_size: 20
components:
  - id: table_driver_list
    type: data_table
    label: 기사 목록
    required: false
    readonly: true
    visible: true
    enabled: true
    options:
      columns: [driver_id, driver_name, phone, driver_code, owner_name, group_name, primary_vehicle_no, account_linked, app_activation_status, status]
      row_click_action: load_detail
    data_binding: drivers.items
    description: 기사 목록 및 연결 상태
  - id: field_driver_name
    type: text_input
    label: 기사명
    required: true
    readonly: false
    visible: true
    enabled: true
    options: {max_length: 50}
    data_binding: driver_detail.driver_name
    description: 기사 이름
  - id: field_phone
    type: phone_input
    label: 휴대폰번호
    required: true
    readonly: false
    visible: true
    enabled: true
    options: {format: KR_MOBILE}
    data_binding: driver_detail.phone
    description: 기사 연락처
  - id: field_driver_code
    type: text_input
    label: 운전자코드
    required: false
    readonly: false
    visible: true
    enabled: true
    options: {max_length: 30, pattern: '^[A-Z0-9]{1,20}$', description: 'eTAS 전송용 ([별표 2] 참조)'}
    data_binding: driver_detail.driver_code
    description: eTAS 전송용 운전자코드 ([별표 2] 참조, 최대 20자리)
  - id: field_owner_id
    type: async_select
    label: Owner
    required: true
    readonly: false
    visible: true
    enabled: true
    options: {entity: owner}
    data_binding: driver_detail.owner_id
    description: 상위 owner
  - id: field_group_id
    type: async_select
    label: Group
    required: true
    readonly: false
    visible: true
    enabled: true
    options: {entity: group, depends_on: owner_id}
    data_binding: driver_detail.group_id
    description: 소속 그룹
  - id: field_primary_vehicle_no
    type: text
    label: 기본 차량
    required: false
    readonly: true
    visible: true
    enabled: false
    options: {}
    data_binding: driver_detail.primary_vehicle_no
    description: 현재 기본 매핑 차량
  - id: field_account_linked
    type: badge
    label: 계정 연결 여부
    required: false
    readonly: true
    visible: true
    enabled: false
    options: {values: [YES, NO]}
    data_binding: driver_detail.account_linked
    description: Driver-Account 연결 여부
  - id: button_link_account
    type: button
    label: 계정 연결
    required: false
    readonly: false
    visible: true
    enabled: true
    options: {variant: secondary}
    data_binding: null
    description: Driver-Account Linking 모달 호출
  - id: button_save_driver
    type: button
    label: 저장
    required: false
    readonly: false
    visible: true
    enabled: true
    options: {variant: primary}
    data_binding: null
    description: 기사 정보 저장
actions:
  - trigger: on_load
    target: driver_page
    behavior: fetch_driver_list
    api_ref: GET /api/admin/drivers
    success_result: 기사 목록 표시
    error_result: 기사 목록 조회 실패
  - trigger: on_click_row
    target: table_driver_list
    behavior: fetch_driver_detail
    api_ref: GET /api/admin/drivers/{driverId}
    success_result: 기사 상세 표시
    error_result: 기사 상세 조회 실패
  - trigger: on_click_link_account
    target: button_link_account
    behavior: open_linking_modal
    api_ref: null
    success_result: 계정 연결 모달 표시
    error_result: null
  - trigger: on_click_save_driver
    target: button_save_driver
    behavior: create_or_update_driver
    api_ref: POST /api/admin/drivers or PUT /api/admin/drivers/{driverId}
    success_result: 기사 저장 후 목록 갱신
    error_result: 기사 저장 실패
states:
  initial: 목록 중심 화면
  normal: 기사 상세와 계정 연결 상태를 표시
  loading: 목록/상세 별도 로딩
  empty: 등록된 기사가 없습니다.
  error: 기사 데이터를 불러오지 못했습니다.
  no_permission: 기사 조회 권한이 없습니다.
  conditional:
    - condition: driver_detail.account_linked == NO
      result: 앱 초대 전 연결 필요 경고를 표시한다.
validation:
  driver_name:
    rules: [필수 입력, 2자 이상 50자 이하]
    messages:
      required: 기사명을 입력해 주세요.
  phone:
    rules: [필수 입력, 대한민국 휴대폰 형식]
    messages:
      required: 휴대폰번호를 입력해 주세요.
      format: 휴대폰번호 형식을 확인해 주세요.
  owner_id:
    rules: [필수 선택]
    messages:
      required: Owner를 선택해 주세요.
  group_id:
    rules: [필수 선택]
    messages:
      required: Group을 선택해 주세요.
permissions:
  view: [Installer, Operator, Admin]
  create: [Installer, Operator, Admin]
  update: [Operator, Admin]
  delete: []
  field_control:
    - field: primary_vehicle_no
      editable_roles: []
    - field: account_linked
      editable_roles: []
api:
  - method: GET
    endpoint: /api/admin/drivers
    purpose: 기사 목록 조회
    request:
      query: {keyword: string, owner_id: string, group_id: string, account_linked: string, status: string, page: number, size: number, sort: string}
    response: {items: array, page_info: object}
  - method: GET
    endpoint: /api/admin/drivers/{driverId}
    purpose: 기사 상세 조회
    request: {path: {driverId: string}}
    response: {driver: object}
  - method: POST
    endpoint: /api/admin/drivers
    purpose: 기사 생성
    request:
      body: {driver_name: string, phone: string, driver_code: string, owner_id: string, group_id: string}
    response: {driver_id: string}
  - method: PUT
    endpoint: /api/admin/drivers/{driverId}
    purpose: 기사 수정
    request:
      path: {driverId: string}
      body: {driver_name: string, phone: string, driver_code: string, group_id: string, status: string}
    response: {driver_id: string}
navigation_rules:
  entry_points:
    - 메뉴 Drivers 클릭
    - 차량 상세의 기본 기사 링크 클릭
  exits:
    - Driver-Account Linking 모달
    - Quick Mapping 화면
    - App Activation 화면
ui_text:
  title: 기사 관리
  empty_message: 등록된 기사가 없습니다.
  error_message: 기사 데이터를 불러오지 못했습니다.
  toast:
    save_success: 기사 정보가 저장되었습니다.
    save_error: 기사 저장에 실패했습니다.
  confirm_message: {}
test_points:
  - account_linked 필터가 정상 동작하는가
  - 계정 미연결 기사에 경고 표시가 노출되는가
  - 기사 저장 후 기본 차량과 연결 상태 표시는 읽기 전용인가
```

### 2.9 Driver-Account Linking 모달
```yaml
screen:
  id: SCR-LINK-001
  name: Driver-Account Linking 모달
  service: Installer Admin Web
  domain: Account Linking
  route: modal://drivers/link-account
  menu_path: Drivers > 계정 연결
  roles: [Installer, Operator, Admin]
  purpose: driver 엔터티와 앱 로그인 계정을 연결하거나 새로 생성한다.
layout:
  type: modal
  platform: web-desktop
  navigation:
    header: modal_header
    sidebar: none
    tabs: [search_existing_account, create_new_account]
data_scope:
  primary_entity: driver_account_link
  params: [driver_id, keyword]
  filters:
    - id: keyword
      type: search
  sort:
    default: created_at.desc
    allowed: [created_at.desc, user_name.asc]
  pagination:
    enabled: true
    default_page_size: 10
components:
  - id: field_driver_summary
    type: summary_card
    label: 대상 기사 정보
    required: false
    readonly: true
    visible: true
    enabled: false
    options: {fields: [driver_name, phone, owner_name, group_name]}
    data_binding: target_driver
    description: 계정 연결 대상 기사 요약
  - id: search_existing_account
    type: search_input
    label: 기존 계정 검색
    required: false
    readonly: false
    visible: true
    enabled: true
    options: {debounce_ms: 300}
    data_binding: query.keyword
    description: 이름/휴대폰번호 기준 앱 계정 검색
  - id: table_account_candidates
    type: data_table
    label: 기존 계정 후보
    required: false
    readonly: true
    visible: true
    enabled: true
    options:
      columns: [user_id, user_type, user_name, phone, owner_name, app_activation_status]
      selectable: single
    data_binding: candidate_accounts.items
    description: 연결 가능한 기존 계정 후보
  - id: field_new_user_type
    type: radio_group
    label: 생성할 계정 유형
    required: true
    readonly: false
    visible: true
    enabled: true
    options: {values: [DRIVER, MEMBER]}
    data_binding: form.user_type
    description: 새 계정 생성 시 유형 선택
  - id: field_new_user_name
    type: text_input
    label: 계정 사용자명
    required: true
    readonly: false
    visible: true
    enabled: true
    options: {max_length: 50}
    data_binding: form.user_name
    description: 새 계정 이름
  - id: field_new_phone
    type: phone_input
    label: 계정 휴대폰번호
    required: true
    readonly: false
    visible: true
    enabled: true
    options: {format: KR_MOBILE}
    data_binding: form.phone
    description: 로그인/초대용 번호
  - id: field_invite_now
    type: checkbox
    label: 연결 후 즉시 앱 초대 발송
    required: false
    readonly: false
    visible: true
    enabled: true
    options: {default: true}
    data_binding: form.invite_now
    description: 새 계정 생성 후 바로 초대 여부
  - id: button_link_existing
    type: button
    label: 기존 계정 연결
    required: false
    readonly: false
    visible: true
    enabled: true
    options: {variant: secondary}
    data_binding: null
    description: 선택한 기존 계정 연결
  - id: button_create_and_link
    type: button
    label: 새 계정 생성 후 연결
    required: false
    readonly: false
    visible: true
    enabled: true
    options: {variant: primary}
    data_binding: null
    description: 새 앱 계정 생성 및 연결
actions:
  - trigger: on_search_existing_account
    target: search_existing_account
    behavior: fetch_account_candidates
    api_ref: GET /api/admin/accounts/candidates
    success_result: 기존 계정 후보 목록 표시
    error_result: 후보 조회 실패
  - trigger: on_click_link_existing
    target: button_link_existing
    behavior: link_existing_account
    api_ref: POST /api/admin/drivers/{driverId}/link-account
    success_result: 기사 계정 연결이 완료되었습니다.
    error_result: 계정 연결에 실패했습니다.
  - trigger: on_click_create_and_link
    target: button_create_and_link
    behavior: create_account_and_link
    api_ref: POST /api/admin/drivers/{driverId}/create-linked-account
    success_result: 새 계정 생성 및 연결이 완료되었습니다.
    error_result: 새 계정 생성 또는 연결에 실패했습니다.
states:
  initial: 대상 기사 요약과 빈 검색/생성 폼 표시
  normal: 기존 계정 연결 또는 신규 생성/연결 가능
  loading: 검색 또는 저장 중 버튼 로딩 표시
  empty: 연결 가능한 기존 계정이 없습니다.
  error: 계정 연결 처리 중 오류가 발생했습니다.
  no_permission: 계정 연결 권한이 없습니다.
  conditional:
    - condition: target_driver.account_linked == YES
      result: 기존 연결 계정 정보와 재연결 경고 문구를 표시한다.
validation:
  user_type:
    rules: [신규 생성 시 필수]
    messages:
      required: 계정 유형을 선택해 주세요.
  user_name:
    rules: [신규 생성 시 필수, 2자 이상 50자 이하]
    messages:
      required: 계정 사용자명을 입력해 주세요.
  phone:
    rules: [신규 생성 시 필수, 대한민국 휴대폰 형식]
    messages:
      required: 계정 휴대폰번호를 입력해 주세요.
      format: 휴대폰번호 형식을 확인해 주세요.
permissions:
  view: [Installer, Operator, Admin]
  create: [Installer, Operator, Admin]
  update: [Installer, Operator, Admin]
  delete: []
  field_control:
    - field: user_type
      editable_roles: [Installer, Operator, Admin]
api:
  - method: GET
    endpoint: /api/admin/accounts/candidates
    purpose: 연결 가능한 기존 앱 계정 검색
    request:
      query: {keyword: string, owner_id: string, group_id: string}
    response: {items: array, page_info: object}
  - method: POST
    endpoint: /api/admin/drivers/{driverId}/link-account
    purpose: 기존 계정 연결
    request:
      path: {driverId: string}
      body: {user_id: string}
    response: {driver_id: string, user_id: string, linked: boolean}
  - method: POST
    endpoint: /api/admin/drivers/{driverId}/create-linked-account
    purpose: 새 계정 생성 및 연결
    request:
      path: {driverId: string}
      body: {user_type: string, user_name: string, phone: string, invite_now: boolean}
    response: {driver_id: string, user_id: string, invite_status: string}
navigation_rules:
  entry_points:
    - 기사 상세 화면의 계정 연결 버튼
    - App Activation 화면의 미연결 기사 항목 클릭
  exits:
    - 기사 상세 화면으로 복귀
    - App Activation 화면 갱신
ui_text:
  title: Driver 계정 연결
  empty_message: 연결 가능한 기존 계정이 없습니다.
  error_message: 계정 연결 처리 중 오류가 발생했습니다.
  toast:
    link_success: 기사와 앱 계정 연결이 완료되었습니다.
    create_link_success: 새 계정 생성 및 연결이 완료되었습니다.
    link_error: 계정 연결에 실패했습니다.
  confirm_message:
    relink_warning: 기존 연결 계정을 변경하시겠습니까?
test_points:
  - 기존 계정 검색과 신규 계정 생성 흐름이 모두 동작하는가
  - invite_now 선택 시 앱 초대 상태가 바로 생성되는가
  - 이미 연결된 기사 재연결 시 확인창이 표시되는가
```

### 2.10 Quick Mapping
```yaml
screen:
  id: SCR-MAP-001
  name: Quick Mapping
  service: Installer Admin Web
  domain: Mapping Management
  route: /mappings/quick
  menu_path: Mappings > Quick Mapping
  roles: [Installer, Operator, Admin]
  purpose: 현장에서 DTG-차량-기사-계정을 빠르게 연결하고 바로 검증까지 이어간다.
layout:
  type: wizard
  platform: web-desktop
  navigation:
    header: global_header
    sidebar: main_sidebar
    tabs: [select_vehicle, select_driver, select_device, review]
data_scope:
  primary_entity: mapping_bundle
  params: [owner_id, group_id, vehicle_id, driver_id, device_id]
  filters:
    - id: owner_id
      type: async_select
    - id: group_id
      type: async_select
  sort:
    default: recent_used.desc
    allowed: [recent_used.desc, created_at.desc]
  pagination:
    enabled: true
    default_page_size: 10
components:
  - id: select_owner
    type: async_select
    label: Owner 선택
    required: true
    readonly: false
    visible: true
    enabled: true
    options: {entity: owner}
    data_binding: form.owner_id
    description: 매핑 기준 owner
  - id: select_group
    type: async_select
    label: Group 선택
    required: true
    readonly: false
    visible: true
    enabled: true
    options: {entity: group, depends_on: owner_id}
    data_binding: form.group_id
    description: 매핑 기준 그룹
  - id: select_vehicle
    type: async_search_select
    label: 차량 선택
    required: true
    readonly: false
    visible: true
    enabled: true
    options: {entity: vehicle, create_inline: true}
    data_binding: form.vehicle_id
    description: 기존 차량 선택 또는 즉시 등록
  - id: select_driver
    type: async_search_select
    label: 기사 선택
    required: true
    readonly: false
    visible: true
    enabled: true
    options: {entity: driver, create_inline: true}
    data_binding: form.driver_id
    description: 기존 기사 선택 또는 즉시 등록
  - id: select_device
    type: async_search_select
    label: DTG 선택
    required: true
    readonly: false
    visible: true
    enabled: true
    options: {entity: device, only_unmapped_or_reassignable: true}
    data_binding: form.device_id
    description: 미매핑 DTG 또는 재할당 가능 장치 선택
  - id: check_link_driver_account
    type: checkbox
    label: 기사 계정 연결 확인 완료
    required: false
    readonly: false
    visible: true
    enabled: true
    options: {default: false}
    data_binding: form.driver_account_checked
    description: 계정 연결 여부 확인 체크
  - id: summary_review
    type: summary_card
    label: 매핑 요약
    required: false
    readonly: true
    visible: true
    enabled: false
    options: {fields: [owner, group, vehicle, driver, device]}
    data_binding: review
    description: 저장 전 매핑 결과 요약
  - id: button_save_mapping
    type: button
    label: 매핑 저장
    required: false
    readonly: false
    visible: true
    enabled: true
    options: {variant: primary}
    data_binding: null
    description: DTG-차량, 차량-기사 매핑 저장
  - id: button_save_and_verify
    type: button
    label: 저장 후 연동 검증
    required: false
    readonly: false
    visible: true
    enabled: true
    options: {variant: primary, emphasized: true}
    data_binding: null
    description: 매핑 저장 후 즉시 검증 센터로 이동
actions:
  - trigger: on_change_owner_group
    target: quick_mapping_page
    behavior: reload_entity_candidates
    api_ref: GET /api/admin/mappings/candidates
    success_result: owner/group 기준 후보 목록 갱신
    error_result: 후보 조회 실패 토스트 표시
  - trigger: on_click_save_mapping
    target: button_save_mapping
    behavior: save_mapping_bundle
    api_ref: POST /api/admin/mappings/bundle
    success_result: 매핑이 저장되었습니다 토스트 표시
    error_result: 매핑 저장 실패 메시지 표시
  - trigger: on_click_save_and_verify
    target: button_save_and_verify
    behavior: save_mapping_and_open_verification
    api_ref: POST /api/admin/mappings/bundle
    success_result: 저장 후 검증 센터로 이동한다.
    error_result: 매핑 저장 실패 시 이동하지 않는다.
states:
  initial: owner/group 선택 전 엔터티 선택 필드는 비활성화한다.
  normal: 선택/요약/저장 가능
  loading: 후보 조회 또는 저장 중 단계별 로딩 표시
  empty: 선택 가능한 차량/기사/DTG가 없습니다.
  error: 매핑 처리 중 오류가 발생했습니다.
  no_permission: 매핑 권한이 없습니다.
  conditional:
    - condition: selected_device.is_already_mapped == true
      result: 재할당 경고와 기존 연결 정보를 표시한다.
    - condition: selected_driver.account_linked == false
      result: Driver-Account Linking 권고 배너를 표시한다.
validation:
  owner_id:
    rules: [필수 선택]
    messages:
      required: Owner를 선택해 주세요.
  group_id:
    rules: [필수 선택]
    messages:
      required: Group을 선택해 주세요.
  vehicle_id:
    rules: [필수 선택]
    messages:
      required: 차량을 선택해 주세요.
  driver_id:
    rules: [필수 선택]
    messages:
      required: 기사를 선택해 주세요.
  device_id:
    rules: [필수 선택]
    messages:
      required: DTG를 선택해 주세요.
permissions:
  view: [Installer, Operator, Admin]
  create: [Installer, Operator, Admin]
  update: [Installer, Operator, Admin]
  delete: []
  field_control:
    - field: device_id
      editable_roles: [Installer, Operator, Admin]
api:
  - method: GET
    endpoint: /api/admin/mappings/candidates
    purpose: owner/group 기준 차량/기사/장치 후보 조회
    request:
      query: {owner_id: string, group_id: string, keyword: string}
    response: {vehicles: array, drivers: array, devices: array}
  - method: POST
    endpoint: /api/admin/mappings/bundle
    purpose: DTG-차량, 차량-기사 묶음 매핑 저장
    request:
      body:
        owner_id: string
        group_id: string
        vehicle_id: string
        driver_id: string
        device_id: string
        driver_account_checked: boolean
    response: {device_vehicle_mapping_id: string, vehicle_driver_mapping_id: string}
navigation_rules:
  entry_points:
    - 메뉴 Mappings > Quick Mapping 클릭
    - 차량/기사/DTG 상세 화면의 매핑 버튼 클릭
  exits:
    - 검증 센터
    - 차량 상세 화면
    - 기사 상세 화면
ui_text:
  title: Quick Mapping
  empty_message: 선택 가능한 엔터티가 없습니다.
  error_message: 매핑 처리 중 오류가 발생했습니다.
  toast:
    save_success: 매핑이 저장되었습니다.
    save_error: 매핑 저장에 실패했습니다.
  confirm_message:
    remap_device: 이 DTG는 다른 차량에 연결되어 있습니다. 재할당하시겠습니까?
test_points:
  - owner/group 선택 전 나머지 선택 필드가 비활성화되는가
  - 재할당 대상 장치 선택 시 경고창이 표시되는가
  - 저장 후 검증 버튼이 검증 센터로 정상 이동시키는가
```

### 2.11 연동 검증 센터
```yaml
screen:
  id: SCR-VER-001
  name: 연동 검증 센터
  service: Installer Admin Web
  domain: Verification
  route: /verification
  menu_path: Verification
  roles: [Installer, Operator, Admin]
  purpose: DTG-차량-기사-앱 상태를 검증하고 PASS/WARNING/FAIL을 판정한다.
layout:
  type: list_detail_split
  platform: web-desktop
  navigation:
    header: global_header
    sidebar: main_sidebar
    tabs: [queue, result_detail]
data_scope:
  primary_entity: verification_result
  params: [keyword, result_status, severity, owner_id, page, size, sort]
  filters:
    - id: keyword
      type: search
    - id: result_status
      type: multi_select
      values: [NOT_CHECKED, CHECKING, PASS, WARNING, FAIL]
    - id: severity
      type: multi_select
      values: [INFO, WARNING, CRITICAL]
    - id: owner_id
      type: async_select
  sort:
    default: checked_at.desc
    allowed: [checked_at.desc, checked_at.asc, result_status.asc]
  pagination:
    enabled: true
    default_page_size: 20
components:
  - id: table_verification_queue
    type: data_table
    label: 검증 대상 목록
    required: false
    readonly: true
    visible: true
    enabled: true
    options:
      columns: [target_type, vehicle_no, device_serial_no, driver_name, gps_received, heartbeat_received, app_login_checked, result_status, severity, checked_at]
      row_click_action: load_result_detail
    data_binding: verifications.items
    description: 최근 검증 결과 목록
  - id: button_run_verification
    type: button
    label: 검증 실행
    required: false
    readonly: false
    visible: true
    enabled: true
    options: {variant: primary}
    data_binding: null
    description: 선택 대상 검증 실행
  - id: detail_result_status
    type: status_badge
    label: 최종 판정
    required: false
    readonly: true
    visible: true
    enabled: false
    options: {values: [PASS, WARNING, FAIL]}
    data_binding: verification_detail.result_status
    description: 최종 검증 결과
  - id: detail_issue_list
    type: checklist
    label: 검증 항목 결과
    required: false
    readonly: true
    visible: true
    enabled: false
    options:
      items: [device_registered, vehicle_mapped, driver_mapped, gps_received, heartbeat_received, app_invited, first_login_done]
    data_binding: verification_detail.check_items
    description: 세부 항목별 검증 성공 여부
  - id: detail_last_received_at
    type: text
    label: 최종 수신 시각
    required: false
    readonly: true
    visible: true
    enabled: false
    options: {format: datetime}
    data_binding: verification_detail.last_received_at
    description: 서버 기준 마지막 수신 시각
  - id: detail_location_summary
    type: text
    label: 최근 위치 정보
    required: false
    readonly: true
    visible: true
    enabled: false
    options: {}
    data_binding: verification_detail.location_summary
    description: 최근 위치 좌표/주소 요약
  - id: button_retry_verification
    type: button
    label: 재검증
    required: false
    readonly: false
    visible: true
    enabled: true
    options: {variant: secondary}
    data_binding: null
    description: 같은 대상 재검증
actions:
  - trigger: on_load
    target: verification_page
    behavior: fetch_verification_list
    api_ref: GET /api/admin/verifications
    success_result: 검증 목록을 표시한다.
    error_result: 검증 목록 조회 실패
  - trigger: on_click_run_verification
    target: button_run_verification
    behavior: request_verification_run
    api_ref: POST /api/admin/verifications/run
    success_result: 검증이 시작되었습니다 토스트를 표시한다.
    error_result: 검증 시작 실패
  - trigger: on_click_row
    target: table_verification_queue
    behavior: fetch_verification_detail
    api_ref: GET /api/admin/verifications/{verificationId}
    success_result: 우측 상세 결과 표시
    error_result: 검증 상세 조회 실패
  - trigger: on_click_retry_verification
    target: button_retry_verification
    behavior: rerun_selected_verification
    api_ref: POST /api/admin/verifications/run
    success_result: 재검증이 시작되었습니다.
    error_result: 재검증 요청 실패
states:
  initial: 최근 검증 결과를 최신순 표시
  normal: 목록과 상세 결과 표시
  loading: 검증 실행 중 체크 상태와 스피너 표시
  empty: 검증 이력이 없습니다.
  error: 검증 데이터를 불러오지 못했습니다.
  no_permission: 검증 조회 권한이 없습니다.
  conditional:
    - condition: verification_detail.result_status == FAIL
      result: CRITICAL 배너와 조치 권장사항을 노출한다.
    - condition: verification_detail.result_status == WARNING
      result: 서비스 활성화 전 확인 필요 안내를 표시한다.
validation:
  selection:
    rules: [검증 실행 시 최소 1개 대상 선택]
    messages:
      required: 검증할 대상을 선택해 주세요.
permissions:
  view: [Installer, Operator, Admin]
  create: [Installer, Operator, Admin]
  update: [Installer, Operator, Admin]
  delete: []
  field_control: []
api:
  - method: GET
    endpoint: /api/admin/verifications
    purpose: 검증 목록 조회
    request:
      query: {keyword: string, result_status: array, severity: array, owner_id: string, page: number, size: number, sort: string}
    response: {items: array, page_info: object}
  - method: GET
    endpoint: /api/admin/verifications/{verificationId}
    purpose: 검증 상세 조회
    request: {path: {verificationId: string}}
    response: {verification: object}
  - method: POST
    endpoint: /api/admin/verifications/run
    purpose: 검증 실행/재실행
    request:
      body: {target_type: string, target_ids: array}
    response: {request_id: string, accepted_count: number}
navigation_rules:
  entry_points:
    - 메뉴 Verification 클릭
    - Quick Mapping 저장 후 연동 검증
    - 대시보드 이상 건수 클릭
  exits:
    - App Activation 화면
    - 차량 상세 화면
    - DTG 상세 화면
ui_text:
  title: 연동 검증 센터
  empty_message: 검증 이력이 없습니다.
  error_message: 검증 데이터를 불러오지 못했습니다.
  toast:
    run_success: 검증이 시작되었습니다.
    run_error: 검증 실행에 실패했습니다.
  confirm_message:
    retry_verification: 같은 대상을 다시 검증하시겠습니까?
test_points:
  - PASS/WARNING/FAIL 판정이 상태 및 심각도로 잘 매핑되는가
  - FAIL 시 CRITICAL 배너와 조치 가이드가 표시되는가
  - 최종 수신 시각과 위치 정보가 결과 상세에 노출되는가
```

### 2.12 App Activation
```yaml
screen:
  id: SCR-ACT-001
  name: 앱 활성화 관리
  service: Installer Admin Web
  domain: App Activation
  route: /app-activation
  menu_path: App Activation
  roles: [Installer, Operator, Admin]
  purpose: 앱 초대, 최초 로그인, 서비스 활성화 완료 여부를 관리한다.
layout:
  type: list_detail_split
  platform: web-desktop
  navigation:
    header: global_header
    sidebar: main_sidebar
    tabs: [list, detail, invite_history]
data_scope:
  primary_entity: activation_case
  params: [keyword, app_activation_status, user_type, owner_id, page, size, sort]
  filters:
    - id: keyword
      type: search
    - id: app_activation_status
      type: multi_select
      values: [NOT_READY, INVITED, FIRST_LOGIN_DONE, LIVE, LOCKED]
    - id: user_type
      type: multi_select
      values: [OWNER, MEMBER, DRIVER]
    - id: owner_id
      type: async_select
  sort:
    default: updated_at.desc
    allowed: [updated_at.desc, first_login_at.desc, invited_at.desc]
  pagination:
    enabled: true
    default_page_size: 20
components:
  - id: table_activation_list
    type: data_table
    label: 앱 활성화 대상 목록
    required: false
    readonly: true
    visible: true
    enabled: true
    options:
      columns: [user_id, user_type, user_name, phone, owner_name, linked_vehicle_count, linked_driver_name, invite_status, first_login_at, app_activation_status]
      row_click_action: load_detail
    data_binding: activations.items
    description: 앱 활성화 상태 목록
  - id: detail_invite_status
    type: status_badge
    label: 초대 상태
    required: false
    readonly: true
    visible: true
    enabled: false
    options: {values: [NOT_READY, INVITED, EXPIRED, ACCEPTED]}
    data_binding: activation_detail.invite_status
    description: 앱 초대 발송/수락 상태
  - id: detail_app_activation_status
    type: status_badge
    label: 앱 활성화 상태
    required: false
    readonly: true
    visible: true
    enabled: false
    options: {values: [NOT_READY, INVITED, FIRST_LOGIN_DONE, LIVE, LOCKED]}
    data_binding: activation_detail.app_activation_status
    description: 서비스 활성화 상태
  - id: detail_first_login_at
    type: text
    label: 최초 로그인 시각
    required: false
    readonly: true
    visible: true
    enabled: false
    options: {format: datetime}
    data_binding: activation_detail.first_login_at
    description: 첫 로그인 발생 시각
  - id: detail_linked_assets
    type: summary_card
    label: 연결 자산 요약
    required: false
    readonly: true
    visible: true
    enabled: false
    options: {fields: [owner_name, group_name, vehicle_list, driver_name, device_serial_no]}
    data_binding: activation_detail.linked_assets
    description: 앱 로그인 후 보여줄 연결 자산
  - id: button_send_invite
    type: button
    label: 앱 초대 발송
    required: false
    readonly: false
    visible: true
    enabled: true
    options: {variant: primary}
    data_binding: null
    description: SMS 링크 또는 임시 비밀번호 발송
  - id: button_resend_invite
    type: button
    label: 초대 재발송
    required: false
    readonly: false
    visible: true
    enabled: true
    options: {variant: secondary}
    data_binding: null
    description: 기존 초대 재발송
  - id: button_mark_live
    type: button
    label: 서비스 활성화 완료 처리
    required: false
    readonly: false
    visible: true
    enabled: true
    options: {variant: primary}
    data_binding: null
    description: FIRST_LOGIN_DONE 이후 LIVE 전환
actions:
  - trigger: on_load
    target: activation_page
    behavior: fetch_activation_list
    api_ref: GET /api/admin/app-activations
    success_result: 앱 활성화 목록 표시
    error_result: 앱 활성화 목록 조회 실패
  - trigger: on_click_row
    target: table_activation_list
    behavior: fetch_activation_detail
    api_ref: GET /api/admin/app-activations/{activationId}
    success_result: 활성화 상세 표시
    error_result: 상세 조회 실패
  - trigger: on_click_send_invite
    target: button_send_invite
    behavior: send_app_invite
    api_ref: POST /api/admin/app-activations/{activationId}/invite
    success_result: 앱 초대를 발송했습니다 토스트 표시
    error_result: 앱 초대 발송 실패
  - trigger: on_click_resend_invite
    target: button_resend_invite
    behavior: resend_app_invite
    api_ref: POST /api/admin/app-activations/{activationId}/resend-invite
    success_result: 앱 초대를 재발송했습니다.
    error_result: 앱 초대 재발송 실패
  - trigger: on_click_mark_live
    target: button_mark_live
    behavior: mark_service_live
    api_ref: POST /api/admin/app-activations/{activationId}/mark-live
    success_result: 서비스 활성화 완료 처리되었습니다.
    error_result: 서비스 활성화 완료 처리 실패
states:
  initial: 최신 업데이트 순 목록 표시
  normal: 초대 상태와 활성화 상태를 조회/처리 가능
  loading: 목록 또는 액션 버튼 로딩 표시
  empty: 앱 활성화 대상이 없습니다.
  error: 앱 활성화 데이터를 불러오지 못했습니다.
  no_permission: 앱 활성화 관리 권한이 없습니다.
  conditional:
    - condition: activation_detail.app_activation_status == NOT_READY
      result: 계정 연결 또는 검증 선행 필요 경고를 표시한다.
    - condition: activation_detail.app_activation_status == FIRST_LOGIN_DONE
      result: LIVE 완료 처리 버튼을 활성화한다.
    - condition: activation_detail.app_activation_status == LIVE
      result: 완료 배지를 강조하고 재초대 버튼을 숨긴다.
validation:
  invite:
    rules: [phone 존재, 계정 연결 완료, 최소 1개 자산 연결]
    messages:
      missing_phone: 앱 초대를 위해 휴대폰번호가 필요합니다.
      missing_link: 앱 초대 전 계정 연결을 완료해 주세요.
      missing_asset: 연결된 차량 또는 기사 정보가 필요합니다.
permissions:
  view: [Installer, Operator, Admin]
  create: [Installer, Operator, Admin]
  update: [Installer, Operator, Admin]
  delete: []
  field_control:
    - field: app_activation_status
      editable_roles: [Operator, Admin]
api:
  - method: GET
    endpoint: /api/admin/app-activations
    purpose: 앱 활성화 목록 조회
    request:
      query: {keyword: string, app_activation_status: array, user_type: array, owner_id: string, page: number, size: number, sort: string}
    response: {items: array, page_info: object}
  - method: GET
    endpoint: /api/admin/app-activations/{activationId}
    purpose: 앱 활성화 상세 조회
    request: {path: {activationId: string}}
    response: {activation: object, invite_history: array}
  - method: POST
    endpoint: /api/admin/app-activations/{activationId}/invite
    purpose: 앱 초대 발송
    request: {path: {activationId: string}, body: {method: string}}
    response: {invite_status: string, invited_at: datetime}
  - method: POST
    endpoint: /api/admin/app-activations/{activationId}/resend-invite
    purpose: 앱 초대 재발송
    request: {path: {activationId: string}}
    response: {invite_status: string, invited_at: datetime}
  - method: POST
    endpoint: /api/admin/app-activations/{activationId}/mark-live
    purpose: 서비스 활성화 완료 처리
    request: {path: {activationId: string}}
    response: {app_activation_status: LIVE}
navigation_rules:
  entry_points:
    - 메뉴 App Activation 클릭
    - 검증 센터 WARNING/PASS 상세에서 활성화 이동
  exits:
    - Driver-Account Linking 모달
    - Owner/기사/차량 상세 화면
ui_text:
  title: 앱 활성화 관리
  empty_message: 앱 활성화 대상이 없습니다.
  error_message: 앱 활성화 데이터를 불러오지 못했습니다.
  toast:
    invite_success: 앱 초대를 발송했습니다.
    resend_success: 앱 초대를 재발송했습니다.
    live_success: 서비스 활성화 완료 처리되었습니다.
  confirm_message:
    resend_invite: 앱 초대를 다시 발송하시겠습니까?
    mark_live: 서비스 활성화 완료로 처리하시겠습니까?
test_points:
  - NOT_READY 상태에서 초대 발송이 차단되는가
  - FIRST_LOGIN_DONE 상태에서만 LIVE 완료 처리 가능한가
  - LIVE 상태에서는 재초대 버튼이 숨겨지는가
```

### 2.13 시스템 사용자/권한 관리
```yaml
screen:
  id: SCR-SYS-001
  name: 시스템 사용자/권한 관리
  service: Installer Admin Web
  domain: System Administration
  route: /system/users
  menu_path: System > Users & Roles
  roles: [Admin]
  purpose: Installer Admin 내부 사용자와 역할을 관리한다.
layout:
  type: list_detail_split
  platform: web-desktop
  navigation:
    header: global_header
    sidebar: main_sidebar
    tabs: [users, roles]
data_scope:
  primary_entity: admin_user
  params: [keyword, role, status, page, size, sort]
  filters:
    - id: keyword
      type: search
    - id: role
      type: multi_select
      values: [Installer, Operator, Admin]
    - id: status
      type: select
      values: [ALL, ACTIVE, INACTIVE, LOCKED]
  sort:
    default: created_at.desc
    allowed: [created_at.desc, user_name.asc]
  pagination:
    enabled: true
    default_page_size: 20
components:
  - id: table_admin_users
    type: data_table
    label: 시스템 사용자 목록
    required: false
    readonly: true
    visible: true
    enabled: true
    options:
      columns: [user_id, user_name, email, role, status, last_login_at, created_at]
      row_click_action: load_detail
    data_binding: admin_users.items
    description: 내부 관리자/설치기사 계정 목록
  - id: field_user_name
    type: text_input
    label: 사용자명
    required: true
    readonly: false
    visible: true
    enabled: true
    options: {max_length: 50}
    data_binding: user_detail.user_name
    description: 내부 사용자 이름
  - id: field_email
    type: email_input
    label: 이메일
    required: true
    readonly: false
    visible: true
    enabled: true
    options: {max_length: 100}
    data_binding: user_detail.email
    description: 로그인 이메일
  - id: field_role
    type: select
    label: 역할
    required: true
    readonly: false
    visible: true
    enabled: true
    options: {values: [Installer, Operator, Admin]}
    data_binding: user_detail.role
    description: 시스템 역할
  - id: field_status
    type: select
    label: 상태
    required: true
    readonly: false
    visible: true
    enabled: true
    options: {values: [ACTIVE, INACTIVE, LOCKED]}
    data_binding: user_detail.status
    description: 계정 상태
  - id: button_save_user
    type: button
    label: 저장
    required: false
    readonly: false
    visible: true
    enabled: true
    options: {variant: primary}
    data_binding: null
    description: 내부 사용자 저장
actions:
  - trigger: on_load
    target: system_user_page
    behavior: fetch_admin_users
    api_ref: GET /api/admin/system/users
    success_result: 사용자 목록 표시
    error_result: 사용자 목록 조회 실패
  - trigger: on_click_row
    target: table_admin_users
    behavior: fetch_admin_user_detail
    api_ref: GET /api/admin/system/users/{userId}
    success_result: 상세 표시
    error_result: 상세 조회 실패
  - trigger: on_click_save_user
    target: button_save_user
    behavior: create_or_update_admin_user
    api_ref: POST /api/admin/system/users or PUT /api/admin/system/users/{userId}
    success_result: 사용자 저장 후 목록 갱신
    error_result: 사용자 저장 실패
states:
  initial: Admin만 접근 가능
  normal: 목록/상세/역할 편집 가능
  loading: 목록/상세 로더 표시
  empty: 등록된 시스템 사용자가 없습니다.
  error: 시스템 사용자 정보를 불러오지 못했습니다.
  no_permission: 시스템 관리 권한이 없습니다.
  conditional: []
validation:
  user_name:
    rules: [필수 입력, 2자 이상 50자 이하]
    messages:
      required: 사용자명을 입력해 주세요.
  email:
    rules: [필수 입력, 이메일 형식, 중복 불가]
    messages:
      required: 이메일을 입력해 주세요.
      format: 이메일 형식을 확인해 주세요.
      duplicate: 이미 사용 중인 이메일입니다.
  role:
    rules: [필수 선택]
    messages:
      required: 역할을 선택해 주세요.
permissions:
  view: [Admin]
  create: [Admin]
  update: [Admin]
  delete: []
  field_control:
    - field: role
      editable_roles: [Admin]
    - field: status
      editable_roles: [Admin]
api:
  - method: GET
    endpoint: /api/admin/system/users
    purpose: 시스템 사용자 목록 조회
    request:
      query: {keyword: string, role: array, status: string, page: number, size: number, sort: string}
    response: {items: array, page_info: object}
  - method: POST
    endpoint: /api/admin/system/users
    purpose: 시스템 사용자 생성
    request:
      body: {user_name: string, email: string, role: string, status: string}
    response: {user_id: string}
navigation_rules:
  entry_points:
    - 메뉴 System > Users & Roles 클릭
  exits:
    - 없음
ui_text:
  title: 시스템 사용자/권한 관리
  empty_message: 등록된 시스템 사용자가 없습니다.
  error_message: 시스템 사용자 정보를 불러오지 못했습니다.
  toast:
    save_success: 시스템 사용자 정보가 저장되었습니다.
    save_error: 시스템 사용자 저장에 실패했습니다.
  confirm_message: {}
test_points:
  - Admin 외 역할 접근 시 no_permission 처리되는가
  - 역할/상태 변경이 정상 저장되는가
  - 이메일 중복 검증이 동작하는가
```

---

## 3. 변경 이력 (Changelog)

- **v1.1 (2026-03-25):**
  - Backend 문서 참조: eTAS 전송 기준정보 필드 보강
  - 차량 목록: `vehicle_type` 컬럼 추가
  - 차량 상세: `vehicle_type` 필드 추가 (eTAS 전송용)
  - 차량번호 검증: `[별표 4]` 차량번호 형식 검증 안내 추가
  - VIN 검증: 17자리 영문/숫자 패턴 명시 (I, O, Q 제외)
  - DTG 목록/상세: `PENDING_ACTIVATION`, `ACTIVE` 상태값 추가
  - DTG 장치 상태: 상태값 의미 설명 추가
  - Driver 목록: `driver_code` 컬럼 확인 (기존 필드 유지)
  - Driver 상세: `driver_code` 필드 설명 보강

- **v1.0 (2026-03-25):**
  - 기존 화면설계서(installer_admin_screen_spec_v1.md)를 리네이밍 및 문서 구조 개선
  - 문서 명명 규칙(TrackCar Installer Admin *.md) 적용
  - 메타데이터(작성일, 버전, 상태) 추가
  - 화면 번호 체계(2.1~2.13) 적용

---

## 4. 참고 문서

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
