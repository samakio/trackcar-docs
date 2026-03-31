# TrackCar Mobile App - Replit 개발 가이드

## 1. 문서 목적

본 문서는 Replit/외부 프론트 개발자가 TrackCar Mobile App을 구현할 때 필요한 **최신 화면 구조, API 범위, 역할 규칙**을 빠르게 이해하도록 돕는 가이드다.

기준 문서:
- `TrackCar Mobile App 요구사항 및 기능 명세서.md`
- `TrackCar Mobile App 프로세스 및 IA 설계서.md`
- `TrackCar Mobile App API Specification.md`
- `TrackCar Mobile App 화면 상세 설계서.md`

---

## 2. 핵심 개념

- Mobile App은 **고객사 운영 앱**이다.
- Internal Admin Web는 **설치/개통/연동 검증용 내부 시스템**이다.
- Mobile App은 설치가 끝난 뒤의 **운영 중 조회/관리**에 집중한다.

### 역할

| 역할 | 설명 |
|------|------|
| OWNER | 고객사 관리자 |
| STAFF | 고객사 담당자 |

---

## 3. 메뉴 구조

### 공통 하단 탭
- Home
- Vehicles
- Trips
- Alerts
- My

### OWNER 전용 탭
- 운영

### 운영 메뉴 하위
- 담당자 관리
- 그룹 관리
- 알림 관리

---

## 4. 홈 대시보드 방향

### Home에 보여줄 것
- 전체차량
- 운행중
- 정차중
- 오프라인
- 전체차량위치 버튼

### Home에서 빼는 것
- 조치 필요 차량 리스트
- 최근 알림 리스트
- 빠른 이동 링크

설명:
Home은 **관제 요약** 역할만 담당한다.

---

## 5. 전체차량위치 화면

새 화면: `/vehicles/map`

목적:
- 조직 내 차량 전체를 지도에서 확인

기능:
- marker 표시
- 차량번호/기사명 검색
- 그룹 필터
- marker 선택 시 차량 상세 이동

설명:
- 버튼은 Home에 두고,
- 화면은 Vehicles 도메인 하위에 둔다.

---

## 6. 유지할 핵심 화면

### Vehicles
- 차량 목록
- 차량 상세
- 전체차량위치

### Trips
- 운행 이력 목록
- 운행 상세

### Alerts
- 알림 목록
- 알림 상세

### My
- 내 정보 조회
- 알림 설정

---

## 7. 운영 메뉴 화면

### 7.1 담당자 관리
기능:
- 목록
- 추가
- 수정
- 활성/비활성

### 7.2 그룹 관리
기능:
- 목록
- 추가
- 수정
- 삭제

### 7.3 알림 관리
현재 포함:
- 시동 ON/OFF 알림
- 공회전 알림

후속:
- 위치 알림

설정 단위:
- 차량 선택
- 알림 on/off 저장

---

## 8. API 범위 가이드

### 바로 구현 가능한 조회 중심 API
- `GET /v1/mobile/dashboard`
- `GET /v1/mobile/vehicles`
- `GET /v1/mobile/vehicles/{vehicleId}`
- `GET /v1/mobile/vehicles/map`
- `GET /v1/mobile/trips`
- `GET /v1/mobile/trips/{tripId}`
- `GET /v1/mobile/alerts`
- `PATCH /v1/mobile/alerts/{alertId}/read`
- `PATCH /v1/mobile/alerts/read-all`
- `GET /v1/mobile/me`
- `PATCH /v1/mobile/me/notification-settings`

### 구현된 운영 API
- `GET/POST/PATCH /v1/mobile/staff-users...`
- `GET/POST/PATCH/DELETE /v1/mobile/groups...`
- `GET/POST/PATCH /v1/mobile/alert-settings...`

### 후속 구현 필요 API
- 위치 알림 관련 API

> 구현 전 `TrackCar Mobile App API Specification.md`의 상태 표시(구현/계획)를 확인할 것.

---

## 9. 구현 우선순위 추천

### 1단계
- Auth
- Home (축소 KPI)
- Vehicles
- Vehicle Detail
- Vehicle Map
- Trips
- Alerts
- My

### 2단계
- 운영 메뉴 진입

### 3단계
- 운영 메뉴(담당자 관리 / 그룹 관리 / 알림 관리)

### 4단계
- 위치 알림

---

## 10. 주의사항

1. `OWNER`, `STAFF`는 API/코드 값으로 유지한다.
2. UI 라벨은 필요 시 “고객사 관리자”, “담당자”로 풀어쓴다.
3. Mobile App에서 설치/연동 검증 화면을 구현하지 않는다.
4. 운영 메뉴는 OWNER 전용이다.
5. `Vehicle Map View`는 Home의 버튼 진입이지만 Vehicles 도메인 하위 화면으로 본다.

---

## 11. 추천 개발 순서

1. 인증/세션
2. 차량 목록/상세
3. 전체차량위치
4. 운행 이력
5. 알림
6. 내 정보
7. 운영 메뉴

---

## 12. 현재 backend 기준 메모

- `GET /v1/mobile/dashboard` 사용 가능
- `GET /v1/mobile/vehicles/map` 사용 가능
- `PATCH /v1/mobile/alerts/read-all`는 organization 범위로 처리되도록 보정됨
- `PATCH /v1/mobile/me/notification-settings`는 실제 `app_user.push_enabled`, `email_enabled`를 업데이트함
- `staff-users`, `groups`, `alert-settings` API는 기본 CRUD/설정 저장 기준으로 사용 가능
