# TrackCar Mobile App 프로세스 및 IA 설계서

- 작성일: 2026-03-31
- 버전: v2.0
- 상태: 재정비 초안

---

## 1. 문서 목적

본 문서는 TrackCar Mobile App의 **사용자 흐름, 메뉴 구조, 정보 구조(IA)** 를 정의한다.

기준:
- Mobile App은 고객사 운영 사용자를 위한 관제/조회/운영 앱이다.
- 설치/개통/연동 검증은 Internal Admin Web 영역이다.
- Mobile App은 운영 중 상태 조회와 운영 설정에 집중한다.

---

## 2. 역할과 시스템 경계

| 역할 | 설명 | Mobile App |
|------|------|------------|
| OWNER | 고객사 관리자 | 전체 조회 + 운영 메뉴 접근 |
| STAFF | 고객사 담당자 | 조회 중심 |
| Internal Admin | 설치기사/운영자 | Mobile App 범위 제외 |

### 경계 원칙
- Internal Admin Web: 기준정보 생성, 매핑, 연동 검증
- Mobile App: 차량 운영 상태 확인, 이력 조회, 알림 확인, 운영 설정

---

## 3. 상위 사용자 흐름

### 3.1 OWNER 흐름

```text
로그인
  → Home
    → 전체차량위치
    → Vehicles
    → Trips
    → Alerts
    → 운영
    → My
```

### 3.2 STAFF 흐름

```text
로그인
  → Home
    → 전체차량위치
    → Vehicles
    → Trips
    → Alerts
    → My
```

---

## 4. 하단 탭 구조

### 공통 탭
- Home
- Vehicles
- Trips
- Alerts
- My

### OWNER 전용 탭
- 운영

> `운영`은 기존 `Workspace`의 역할을 대체하는 사용자 노출 이름이다.

---

## 5. 메뉴 구조도

### 5.1 전체 구조

```text
App
├─ Auth
│  ├─ Splash
│  ├─ Login
│  └─ Initial Password Change (if needed)
├─ Main
│  ├─ Home
│  ├─ Vehicles
│  ├─ Trips
│  ├─ Alerts
│  ├─ 운영 (OWNER only)
│  └─ My
└─ Common Modal / Bottom Sheet
   ├─ Search
   ├─ Filter
   ├─ Select Vehicle
   ├─ Select Staff/Group
   └─ Confirm / Result
```

### 5.2 Home

```text
Home
├─ KPI: 전체차량
├─ KPI: 운행중
├─ KPI: 정차중
├─ KPI: 오프라인
└─ 전체차량위치 버튼
```

### 5.3 Vehicles

```text
Vehicles
├─ 차량 목록
├─ 전체차량위치
└─ 차량 상세
```

### 5.4 Trips

```text
Trips
├─ 운행 이력 목록
└─ 운행 상세
```

### 5.5 Alerts

```text
Alerts
├─ 알림 목록
└─ 알림 상세
```

### 5.6 운영 (OWNER)

```text
운영
├─ 담당자 관리
├─ 그룹 관리
└─ 알림 관리
```

### 5.7 My

```text
My
└─ 내 정보 / 설정
```

---

## 6. 핵심 프로세스

### 6.1 홈 관제 프로세스

```text
Home 진입
  → KPI 조회
  → 상태 확인
  → 전체차량위치 진입 또는 차량 목록 이동
```

### 6.2 전체차량위치 프로세스

```text
전체차량위치 화면 진입
  → 차량 marker 로딩
  → 차량 검색 / 그룹 필터
  → 특정 차량 선택
  → 차량 상세 이동
```

### 6.3 차량 조회 프로세스

```text
차량 목록
  → 검색 / 필터
  → 차량 선택
  → 차량 상세
```

### 6.4 운행 이력 프로세스

```text
Trips 진입
  → 기간 선택
  → 목록 조회
  → 운행 상세
```

### 6.5 알림 프로세스

```text
Alerts 진입
  → 필터 적용
  → 알림 상세
  → 읽음 처리
```

### 6.6 운영 프로세스 (OWNER)

```text
운영 진입
  → 담당자 관리 또는 그룹 관리 또는 알림 관리 선택
```

### 6.7 담당자 관리 프로세스

```text
운영
  → 담당자 관리
  → 목록 조회
  → 추가 / 수정 / 활성화 / 비활성화
```

### 6.8 그룹 관리 프로세스

```text
운영
  → 그룹 관리
  → 목록 조회
  → 추가 / 수정 / 삭제
```

### 6.9 알림 관리 프로세스

```text
운영
  → 알림 관리
  → 차량 선택
  → 시동 ON/OFF 알림 설정
  → 공회전 알림 설정
  → 저장
```

> 위치 알림은 후속 기능으로 분리한다.

---

## 7. 권한별 메뉴 노출 정책

| 메뉴 | OWNER | STAFF | 비고 |
|------|-------|-------|------|
| Home | O | O | 공통 |
| Vehicles | O | O | 공통 |
| 전체차량위치 | O | O | Home/Vehicle 진입 |
| Trips | O | O | 공통 |
| Alerts | O | O | 공통 |
| 운영 | O | X | OWNER 전용 |
| 담당자 관리 | O | X | OWNER 전용 |
| 그룹 관리 | O | X | OWNER 전용 |
| 알림 관리 | O | X | OWNER 전용 |
| My | O | O | 공통 |

---

## 8. 삭제/축소 대상

기존 문서에 있던 아래 항목은 MVP 기준으로 축소 또는 제외한다.

- Home의 조치 필요 차량 리스트
- Home의 최근 알림 리스트
- Home의 빠른 이동 링크
- Driver Management 독립 메뉴
- Vehicle-Driver Link Management 독립 메뉴
- Geofence / Notification Settings 독립 메뉴

설명:
- 차량-기사 연결은 차량 상세 또는 후속 운영 기능으로 내린다.
- 알림 설정은 `운영 > 알림 관리`로 모은다.
- 위치 알림/Geofence는 후속 범위다.

---

## 9. IA 설계 의도

### 9.1 Home
- 현재 상태 요약
- 깊은 운영 기능은 두지 않음

### 9.2 Vehicles
- 목록 + 지도 + 상세의 3단 구조
- 공간 관제와 상세 조회를 분리

### 9.3 운영
- 조직 운영 기능의 허브
- Home과 분리해서 모바일 첫 화면 복잡도 감소

### 9.4 Alerts
- 알림 이력/대응 중심
- 알림 설정은 운영 메뉴로 분리

---

## 10. MVP 우선순위

### 우선 구현
- Home
- 전체차량위치
- Vehicles
- Vehicle Detail
- Trips
- Alerts
- My

### OWNER 운영 기능
- 담당자 관리
- 그룹 관리
- 알림 관리

### 후속 구현
- 위치 알림
- 고급 권한 위임
- 지오펜스

---

## 11. 참고 문서

- `TrackCar Mobile App 요구사항 및 기능 명세서.md`
- `TrackCar Mobile App API Specification.md`
- `TrackCar Mobile App 화면 상세 설계서.md`
- `TrackCar_Mobile_App_Replit_Guide.md`
- `TrackCar Installer Admin 프로세스 및 IA 설계서.md`
- `TrackCar 플랫폼 아키텍처 상세 설계.md`
