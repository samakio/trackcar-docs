# TrackCar Mobile App 화면 상세 설계서

- 작성일: 2026-03-31
- 버전: v2.0
- 상태: 재정비 초안

---

## 1. 문서 목적

본 문서는 TrackCar Mobile App의 화면을 **기능, 상태, 데이터, 액션, 권한** 중심으로 정의한다.

범위:
- Home
- 전체차량위치
- Vehicles
- Vehicle Detail
- Trips
- Alerts
- 운영 (담당자/그룹/알림 관리)
- My

---

## 2. 공통 규칙

- Mobile App은 고객사 운영 사용자를 위한 앱이다.
- 설치/개통/연동 검증은 Admin Web 범위다.
- Home은 상태 요약에 집중한다.
- 운영 기능은 별도 `운영` 메뉴로 분리한다.
- 위치 알림은 후속 기능이다.

---

## 3. 홈 대시보드

```yaml
screen:
  id: SCR-HOME-001
  name: 홈 대시보드
  route: /home
  roles: [OWNER, STAFF]
  purpose: 현재 차량 운영 상태를 KPI로 요약하고 전체차량위치 화면으로 진입한다.
  api_status: implemented_backend_alignment

layout:
  header: 조직 선택 + 마지막 갱신 시각 + 새로고침
  body: KPI 4개 + 전체차량위치 버튼
  footer: 하단 탭바

components:
  - 전체차량
  - 운행중
  - 정차중
  - 오프라인
  - 전체차량위치 버튼

actions:
  - 화면 진입 -> GET /v1/mobile/dashboard
  - 새로고침 -> GET /v1/mobile/dashboard
  - 전체차량위치 버튼 탭 -> /vehicles/map

states:
  initial: 스켈레톤 KPI 표시
  normal: KPI 4개 표시
  loading: 수동 새로고침 중 인디케이터 표시
  error: 대시보드를 불러오지 못했습니다.

permissions:
  view: OWNER, STAFF
```

---

## 4. 전체차량위치

```yaml
screen:
  id: SCR-VEH-MAP-001
  name: 전체차량위치
  route: /vehicles/map
  roles: [OWNER, STAFF]
  purpose: 조직 내 차량 전체를 지도에서 확인하고 특정 차량 상세로 이동한다.
  api_status: implemented_backend_alignment

layout:
  header: 뒤로가기 + 검색 + 그룹 필터
  body: 지도 + 차량 요약 리스트
  footer: 없음

components:
  - 지도
  - 차량 marker
  - 차량번호 검색
  - 그룹 필터
  - 선택 차량 요약 카드

actions:
  - 화면 진입 -> GET /v1/mobile/vehicles/map
  - 차량 검색 -> GET /v1/mobile/vehicles/map
  - marker 탭 -> 차량 상세 이동

states:
  initial: 지도 로딩
  normal: 차량 marker 표시
  empty: 표시 가능한 차량 위치가 없습니다.
  error: 차량 위치를 불러오지 못했습니다.

permissions:
  view: OWNER, STAFF
```

---

## 5. 차량 목록

```yaml
screen:
  id: SCR-VEH-001
  name: 차량 목록
  route: /vehicles
  roles: [OWNER, STAFF]
  purpose: 차량을 검색, 필터, 정렬하여 조회하고 상세 화면으로 이동한다.

layout:
  header: 제목 + 검색 + 필터
  body: 차량 리스트
  footer: 하단 탭바

components:
  - 검색창
  - 필터 버튼
  - 정렬 선택
  - 차량 카드 리스트

actions:
  - 화면 진입 -> GET /v1/mobile/vehicles
  - 검색 입력 -> GET /v1/mobile/vehicles
  - 필터 적용 -> GET /v1/mobile/vehicles
  - 차량 카드 탭 -> /vehicles/:vehicleId

states:
  initial: 스켈레톤 카드
  normal: 차량 리스트
  loading: infinite scroll loader
  empty: 조건에 맞는 차량이 없습니다.
  error: 차량 목록을 불러오지 못했습니다.

permissions:
  view: OWNER, STAFF
```

---

## 6. 차량 상세

```yaml
screen:
  id: SCR-VEH-002
  name: 차량 상세
  route: /vehicles/:vehicleId
  roles: [OWNER, STAFF]
  purpose: 차량 상태, 현재 위치, 기사 연결, 최근 알림, 최근 운행을 확인한다.

layout:
  header: 뒤로가기 + 차량번호
  body: 상태 요약 + 현재 위치 지도 + 차량 정보 + 기사 정보 + 최근 알림 + 최근 운행
  footer: 없음

components:
  - 상태 요약 카드
  - 최종 수신 시각
  - 현재 위치 지도
  - 차량 정보
  - 담당 기사 정보
  - 최근 알림
  - 최근 운행

actions:
  - 화면 진입 -> GET /v1/mobile/vehicles/{vehicleId}
  - OWNER: 기사 연결 변경 (후속)
  - OWNER: 그룹 변경 (후속)

states:
  initial: 스켈레톤 상세
  normal: 차량 상태와 위치 표시
  empty: 최근 알림/운행 이력 없음
  error: 차량 정보를 불러오지 못했습니다.

permissions:
  view: OWNER, STAFF
  update: OWNER (후속)
```

---

## 7. 운행 이력 목록

```yaml
screen:
  id: SCR-TRIP-001
  name: 운행 이력 목록
  route: /trips
  roles: [OWNER, STAFF]
  purpose: 기간, 차량, 그룹 기준으로 운행 이력을 조회한다.

layout:
  header: 제목 + 기간 선택 + 필터
  body: 요약 카드 + 운행 리스트
  footer: 하단 탭바

components:
  - 기간 선택기
  - 필터 버튼
  - 기간 요약 카드
  - 운행 리스트

actions:
  - 화면 진입 -> GET /v1/mobile/trips
  - 기간 변경 -> GET /v1/mobile/trips
  - 항목 탭 -> /trips/:tripId

states:
  initial: 스켈레톤 리스트
  normal: 요약 + 목록
  empty: 운행 이력이 없습니다.
  error: 운행 이력을 불러오지 못했습니다.

permissions:
  view: OWNER, STAFF
```

---

## 8. 운행 상세

```yaml
screen:
  id: SCR-TRIP-002
  name: 운행 상세
  route: /trips/:tripId
  roles: [OWNER, STAFF]
  purpose: 개별 운행의 경로, 시간, 거리, 이벤트를 조회한다.

layout:
  header: 뒤로가기 + 운행 상세
  body: 운행 요약 + 경로 지도 + 이벤트 타임라인
  footer: 없음

components:
  - 운행 요약 카드
  - 경로 지도
  - 이벤트 타임라인

actions:
  - 화면 진입 -> GET /v1/mobile/trips/{tripId}

states:
  initial: 스켈레톤 상세
  normal: 지도 + 이벤트
  empty: 이벤트가 없습니다.
  error: 운행 상세를 불러오지 못했습니다.

permissions:
  view: OWNER, STAFF
```

---

## 9. 알림 센터

```yaml
screen:
  id: SCR-ALT-001
  name: 알림 센터
  route: /alerts
  roles: [OWNER, STAFF]
  purpose: 차량 이상, 통신, 운영 알림을 조회하고 읽음 처리한다.
  api_status: implemented_backend_alignment

layout:
  header: 제목 + 필터
  body: 알림 리스트
  footer: 하단 탭바

components:
  - 심각도 필터
  - 상태 필터
  - 알림 리스트
  - 모두 확인 처리 버튼 (OWNER)

actions:
  - 화면 진입 -> GET /v1/mobile/alerts
  - 알림 탭 -> /alerts/:alertId
  - 개별 읽음 -> PATCH /v1/mobile/alerts/{alertId}/read
  - 일괄 읽음 -> PATCH /v1/mobile/alerts/read-all

states:
  initial: 스켈레톤 리스트
  normal: 알림 목록
  empty: 표시할 알림이 없습니다.
  error: 알림을 불러오지 못했습니다.

permissions:
  view: OWNER, STAFF
```

> 참고: 알림 상세(`GET /v1/mobile/alerts/{alertId}`)와 일괄 읽음 처리의 backend 계약은 추가 정합화가 필요하다.

---

## 10. 운영 메뉴 (OWNER)

```yaml
screen:
  id: SCR-OPS-001
  name: 운영
  route: /operations
  roles: [OWNER]
  purpose: OWNER가 조직 운영 관리 기능으로 진입한다.

layout:
  header: 제목
  body: 운영 버튼 리스트
  footer: 하단 탭바

components:
  - 담당자 관리 버튼
  - 그룹 관리 버튼
  - 알림 관리 버튼

actions:
  - 담당자 관리 -> /operations/staff
  - 그룹 관리 -> /operations/groups
  - 알림 관리 -> /operations/alerts

states:
  normal: 운영 메뉴 표시
  no_permission: 운영 메뉴 권한이 없습니다.

permissions:
  view: OWNER
```

---

## 11. 담당자 관리

```yaml
screen:
  id: SCR-OPS-STAFF-001
  name: 담당자 관리
  route: /operations/staff
  roles: [OWNER]
  purpose: 고객사 담당자(STAFF) 계정을 관리한다.

core_functions:
  - 담당자 목록 조회
  - 담당자 추가
  - 담당자 수정
  - 활성/비활성

permissions:
  view: OWNER
  update: OWNER
```

---

## 12. 그룹 관리

```yaml
screen:
  id: SCR-OPS-GROUP-001
  name: 그룹 관리
  route: /operations/groups
  roles: [OWNER]
  purpose: 고객사 운영 그룹을 관리한다.

core_functions:
  - 그룹 목록 조회
  - 그룹 추가
  - 그룹 수정
  - 그룹 삭제

permissions:
  view: OWNER
  update: OWNER
```

---

## 13. 알림 관리

```yaml
screen:
  id: SCR-OPS-ALERT-001
  name: 알림 관리
  route: /operations/alerts
  roles: [OWNER]
  purpose: 차량별 알림 수신 설정을 관리한다.

core_functions:
  - 차량 선택
  - 시동 ON/OFF 알림 설정
  - 공회전 알림 설정
  - 저장

future_scope:
  - 위치 알림

permissions:
  view: OWNER
  update: OWNER
```

---

## 14. 내 정보 / 설정

```yaml
screen:
  id: SCR-MY-001
  name: 내 정보 / 설정
  route: /my
  roles: [OWNER, STAFF]
  purpose: 로그인 사용자 정보와 개인 설정을 확인/수정한다.

core_functions:
  - 내 프로필 조회
  - 소속 정보 조회
  - 알림 설정 조회/변경
  - 앱 버전 확인

permissions:
  view: OWNER, STAFF
  update: OWNER, STAFF
```

---

## 15. 테스트 포인트 요약

- Home KPI 4개 값이 정상 표시되는가
- 전체차량위치에서 marker가 뜨는가
- 차량 목록 → 상세 → 운행 이력 이동이 자연스러운가
- Alerts의 개별 읽음/일괄 읽음이 정상 동작하는가
- OWNER에게만 운영 메뉴가 보이는가
- 운영 메뉴에서 담당자/그룹/알림 관리 화면으로 이동하는가
