# TrackCar Mobile App 프로세스 및 IA 설계서

- 작성일: 2026-03-25
- 버전: v1.0
- 상태: 초안

---

## 1. 문서 개요

### 1.1 목적

본 문서는 TrackCar Mobile App의 주요 업무 프로세스와 정보 구조(IA)/메뉴 구조를 정의한다. Owner 중심 self-service 운영 구조와 Staff 조회/알림 중심 구조를 반영한다.

### 1.2 설계 기준

- Internal Admin Web는 내부 운영자 전용으로 분리한다.
- 일반 사용자(owner 포함)는 내부 운영용 Admin Web에 접근하지 않는다.
- Owner는 모바일 앱 내 Workspace에서 담당자, 그룹, 차량-기사 연결을 직접 관리한다.
- Staff는 조회/알림 중심으로 단순하게 사용한다.
- MVP 범위에서는 정식 일자별 배차 대신 차량-기사의 상시 연결 관계만 관리한다.

---

## 2. 역할 및 시스템 경계

### 2.1 참여자(Actor)

| 역할 | 설명 | 시스템 |
|------|------|--------|
| **Internal Admin** | 설치기사, 서비스 운영자, 검증/CS 담당자 | Internal Admin Web |
| **Owner** | 조직 대표 또는 운영 책임자 | Mobile App |
| **Staff** | Owner가 생성한 담당자 계정 | Mobile App |

### 2.2 역할 분리 원칙

| 구분 | Mobile App | Internal Admin Web |
|------|------------|-------------------|
| Owner 계정 관리 | 가능 | 불가/내부 지원용 |
| Staff 계정 관리 | 가능 (Owner만) | 내부 지원용 |
| 그룹 관리 | 가능 (Owner만) | 불필요 |
| 차량-기사 연결 | 가능 (Owner만) | 내부 지원용 |
| 차량 상태 조회 | 가능 | 가능 |
| 장치 개통/인증/검증 | 불가 | 가능 |
| 원천 데이터 강제 보정 | 불가 | 가능 |

---

## 3. 전체 서비스 플로우

### 3.1 상위 서비스 흐름

```
1. Internal Admin가 고객 조직/장치 개통 준비
2. Owner가 모바일 앱 로그인 및 조직 진입
3. Owner가 기본 운영 단위(그룹/담당자/기사) 설정
4. Owner가 차량을 그룹에 배치하고 기사 연결 상태 정리
5. Owner/Staff가 Home에서 차량 상태와 예외 상황 확인
6. 필요 시 Vehicles, Trips, Alerts, Workspace에서 상세 대응
```

### 3.2 모바일 앱 핵심 탐색 흐름

```
Login
  -> Home
    -> Vehicles
    -> Trips
    -> Alerts
    -> Workspace (Owner only)
    -> My
```

---

## 4. 프로세스 정의

### 4.1 Owner 초기 온보딩 프로세스

**목적:** Owner가 모바일 앱에 처음 진입한 뒤 자기 조직 기준으로 운영 가능한 상태를 만든다.

**선행 조건:**
- Internal Admin가 고객 조직 또는 장치 개통을 완료했거나 최소 진입 가능한 상태를 준비함
- Owner 계정이 발급되어 있음

**흐름:**
```
Owner 로그인
  -> 조직 선택 또는 기본 조직 진입
  -> 초기 안내 확인
  -> Home 진입
  -> Workspace로 이동
  -> 그룹 생성/점검
  -> 담당자 생성
  -> 기사 등록
  -> 차량별 그룹/기사 연결 상태 확인
  -> 운영 준비 완료
```

**완료 조건:**
- 최소 1개 그룹이 존재함
- 필요한 담당자 계정이 생성됨
- 차량의 그룹 소속이 설정됨
- 주요 차량의 기사 연결 상태가 정리됨

### 4.2 Staff 계정 생성 프로세스

**목적:** Owner가 조직 내 실무 담당자를 앱에서 직접 생성하고 접근 범위를 부여한다.

**흐름:**
```
Workspace
  -> Staff Management
  -> 담당자 생성
  -> 기본 정보 입력 (이름, 휴대폰, 이메일)
  -> 접근 가능 그룹/차량 설정
  -> 알림 수신 범위 설정
  -> 저장
  -> 계정 활성화 완료
```

**입력 항목:**
- 이름 (2~30자)
- 휴대폰 번호 (KR 형식)
- 이메일 (중복 불가)
- 접근 범위 (전체 그룹 / 선택 그룹)
- 알림 수신 채널 (푸시/이메일)

**처리 규칙:**
- Owner만 Staff를 생성할 수 있다.
- Staff는 동일 조직 범위에서만 생성할 수 있다.
- Staff는 Owner 권한을 가질 수 없다.

### 4.3 그룹 생성 및 차량 배치 프로세스

**목적:** Owner가 조직의 차량 운영 단위를 그룹 기준으로 정리한다.

**흐름:**
```
Workspace
  -> Group Management
  -> 그룹 생성
  -> 그룹명 입력
  -> 저장
  -> 차량 선택
  -> 그룹 연결
  -> 결과 확인
```

**활용 예시:**
- 지역별 그룹 (서울지사, 부산지사)
- 사업장별 그룹 (본사, 지점A)
- 차종별 그룹 (화물차량, 승용차량)
- 업무 성격별 그룹 (배송팀, 영업팀)

### 4.4 기사 등록 및 차량-기사 연결 프로세스

**목적:** Owner가 차량과 기사의 상시 연결 관계를 설정하여 운영 상태를 명확히 관리한다.

**설계 원칙:**
- 본 프로세스는 `일자별 배차`가 아니다.
- 본 프로세스는 `현재 담당 기사 연결 상태 관리`다.
- UI에서는 `담당자 연결 / 미연결` 상태로 표현한다.

**흐름 A: 기사 등록:**
```
Workspace
  -> Driver Management
  -> 기사 등록
  -> 기사 기본정보 입력 (기사명, 연락처, 운전자코드)
  -> 저장
  -> 기사 목록 반영
```

**흐름 B: 차량-기사 연결:**
```
Workspace
  -> Vehicle-Driver Link Management
  -> 차량 선택
  -> 연결 가능한 기사 선택
  -> 연결 저장
  -> 차량 상세/홈 상태 반영
```

**처리 규칙:**
- 한 차량에는 기본 담당 기사 1명을 연결하는 것을 원칙으로 한다.
- 필요 시 추후 확장으로 보조 기사 연결을 고려할 수 있다.
- 기사 미연결 차량은 예외 차량으로 분류한다.
- 차량 상세와 Home KPI에 연결 상태가 반영된다.

**참고:** 운전자코드는 eTAS 전송용 ([별표 2] 참조)이며, 최대 20자리 영문/숫자 형식이다.

### 4.5 할당 상태 기반 관제 홈 조회 프로세스

**목적:** Owner와 Staff가 현재 운영 현황과 예외 상태를 가장 빠르게 파악한다.

**표시 항목:**
- 전체 차량 수
- 실시간 운행 차량 수 (RUNNING)
- 정차 차량 수 (IDLE)
- 오프라인 차량 수 (OFFLINE)
- 경고 차량 수 (WARNING)
- 담당자 연결 차량 수
- 담당자 미연결 차량 수
- 최근 주요 알림

**처리 규칙:**
- Owner는 전체 조직 또는 선택 그룹 기준으로 본다.
- Staff는 권한 범위 내 차량만 본다.
- `담당자 미연결 차량`은 운영 확인이 필요한 예외 항목으로 강조한다.
- `오프라인 차량`, `경고 차량`도 동일하게 예외 항목으로 취급한다.

**후속 액션:**
- 미연결 차량 클릭 → Workspace > Vehicle-Driver Link Management
- 경고 차량 클릭 → Alerts 또는 Vehicle Detail
- 오프라인 차량 클릭 → Vehicle Detail에서 최근 수신 시각 확인

### 4.6 차량 상태 조회 및 상세 확인 프로세스

**목적:** 차량 단위로 현재 상태, 최근 위치, 담당 기사, 최근 알림을 확인한다.

**주요 조회 항목:**
- 차량번호 (vehicle_no, [별표 4] 형식)
- 차량 별칭
- 자동차유형코드 (vehicle_type, eTAS 전송용)
- VIN (17자리)
- 그룹명
- 상태 (REGISTERED, DEVICE_LINKED, DRIVER_LINKED, VERIFIED, ACTIVE, INACTIVE)
- 운행 상태 (RUNNING, IDLE, OFFLINE, WARNING)
- 연결 기사 (driver_code 포함)
- 최근 위치 수신 시각
- 최근 통신/전송 시각

**상태 판단 예시:**
- `RUNNING`: 최근 데이터 기준 운행 중
- `IDLE`: 정차 상태
- `OFFLINE`: 최근 수신 없음
- `WARNING`: 경고 이벤트 존재

### 4.7 운행 이력 조회 프로세스

**목적:** Owner와 Staff가 특정 차량 또는 그룹의 최근 운행 이력을 조회한다.

**주요 조회 항목:**
- 운행 시작/종료 시각
- 운행 시간
- 운행 거리
- 평균 속도, 최고 속도
- 정차 시간
- 연결 기사 (driver_code 포함)
- 경로 요약

**처리 규칙:**
- 빠른 조회: 오늘/어제/최근 7일
- Staff는 허용된 범위만 조회 가능
- 최대 93일까지 조회 가능

### 4.8 Staff 일상 사용 프로세스

**목적:** Staff가 복잡한 관리 기능 없이 필요한 운영 정보만 빠르게 조회한다.

**흐름:**
```
Staff 로그인
  -> Home에서 상태 확인
  -> Vehicles에서 담당 차량 조회
  -> Trips에서 운행 확인
  -> Alerts에서 이상 상황 확인
  -> My에서 본인 알림 설정 변경
```

**특징:**
- Workspace 메뉴 없음
- 생성/수정 기능 거의 없음
- 조회 + 알림 중심 사용

---

## 5. 정보 구조 (IA)

### 5.1 최상위 구조

```
App
├─ Auth
│  ├─ Splash
│  ├─ Login
│  ├─ Organization Select (optional)
│  └─ Initial Setup (optional)
├─ Main
│  ├─ Home
│  ├─ Vehicles
│  ├─ Trips
│  ├─ Alerts
│  ├─ Workspace (Owner only)
│  └─ My
└─ Common Popup / Modal
   ├─ Filter
   ├─ Search
   ├─ Change Group
   ├─ Assign Driver
   ├─ Invite Staff
   └─ Confirm / Result
```

### 5.2 하단 탭 구조

**Owner (6탭):**
- Home
- Vehicles
- Trips
- Alerts
- Workspace
- My

**Staff (5탭):**
- Home
- Vehicles
- Trips
- Alerts
- My

(`Workspace`는 Owner 전용 탭이다.)

---

## 6. 메뉴 구조도

### 6.1 Owner 메뉴 구조

```
Home
├─ Fleet Summary
├─ Assignment Status Summary
├─ Alert Summary
├─ Group Filter
└─ Shortcut to Vehicle Detail

Vehicles
├─ Vehicle List
│  ├─ Search
│  ├─ Filter
│  └─ Sort
├─ Vehicle Detail
│  ├─ Basic Info
│  ├─ Status Info
│  ├─ Device Status
│  ├─ Current Driver (driver_code 포함)
│  ├─ Today's Summary
│  ├─ Recent Alerts
│  ├─ Change Group
│  └─ Change Driver Link
└─ Vehicle Map View (optional MVP+)

Trips
├─ Trip List
│  ├─ By Vehicle
│  ├─ By Date
│  ├─ By Group
│  └─ Search/Filter
└─ Trip Detail
   ├─ Drive Summary
   ├─ Route Summary
   ├─ Event History
   └─ Transmission Status (summary)

Alerts
├─ Alert List
│  ├─ All
│  ├─ Warning
│  ├─ Offline
│  ├─ Geofence
│  └─ Transmission
└─ Alert Detail
   ├─ Alert Info
   ├─ Related Vehicle
   └─ Related Time

Workspace
├─ Staff Management
│  ├─ Staff List
│  ├─ Invite Staff
│  ├─ Staff Detail
│  ├─ Permission Scope
│  └─ Disable Staff
├─ Group Management
│  ├─ Group List
│  ├─ Create Group
│  ├─ Group Detail
│  ├─ Assign Vehicles to Group
│  └─ Group Access Scope
├─ Driver Management
│  ├─ Driver List
│  ├─ Register Driver
│  ├─ Driver Detail
│  └─ Link Vehicle
├─ Vehicle-Driver Link Management
│  ├─ Linked Vehicles
│  ├─ Unlinked Vehicles
│  ├─ Assign Driver
│  └─ Remove Link
└─ Geofence / Notification Settings
   ├─ Entry/Exit Locations
   ├─ Notification Recipients
   └─ Alert Rules Summary

My
├─ My Profile
├─ Organization Info
├─ My Access Scope
├─ Notification Settings
├─ App Settings
└─ Help / Version
```

### 6.2 Staff User 메뉴 구조

```
Home
├─ Fleet Summary (limited scope)
├─ Alert Summary
└─ Shortcut to Vehicle Detail

Vehicles
├─ Vehicle List (limited scope)
└─ Vehicle Detail
   ├─ Basic Info
   ├─ Status Info
   ├─ Device Status
   ├─ Current Driver
   ├─ Today's Summary
   └─ Recent Alerts

Trips
├─ Trip List (limited scope)
└─ Trip Detail

Alerts
├─ Alert List (limited scope)
└─ Alert Detail

My
├─ My Profile
├─ My Access Scope
├─ Notification Settings
├─ App Settings
└─ Help / Version
```

---

## 7. 권한별 메뉴 노출 정책

| 메뉴 | Owner | Staff User | 비고 |
|------|-------|------------|------|
| Home | O | O | Staff는 허용 범위 내 데이터만 조회 |
| Vehicles | O | O | Staff는 편집 불가 |
| Trips | O | O | Staff는 허용 범위 내 조회 |
| Alerts | O | O | Staff는 허용 범위 내 조회/수신 |
| Workspace | O | X | Owner 전용 |
| Staff Management | O | X | Owner 전용 |
| Group Management | O | X | Owner 전용 |
| Driver Management | O | X | Owner 전용 |
| Vehicle-Driver Link Management | O | X | Owner 전용 |
| Geofence / Notification Settings | O | X | Owner 전용 |

---

## 8. 주요 탐색 시나리오

### 8.1 Owner 시나리오

1. Home에서 미연결 차량 수를 확인한다.
2. Workspace > Vehicle-Driver Link Management로 이동한다.
3. 미연결 차량 목록을 확인한다.
4. 차량을 선택해 기사를 연결한다.
5. 다시 Home으로 돌아와 연결 상태를 확인한다.

### 8.2 Staff 시나리오

1. Home에서 경고 차량 수를 확인한다.
2. Alerts로 이동해 관련 알림을 본다.
3. 알림에서 차량 상세로 이동한다.
4. 차량 상태 및 최근 운행요약을 확인한다.

### 8.3 Owner 담당자 관리 시나리오

1. Workspace > Staff Management로 이동한다.
2. 담당자를 초대한다.
3. 접근 그룹 또는 차량 범위를 지정한다.
4. 담당자는 앱 로그인 후 허용 범위의 정보만 조회한다.

---

## 9. 상태 기반 예외 프로세스

### 9.1 담당자 미연결 차량 대응 프로세스

**흐름:**
```
Home 또는 Vehicles
  -> 미연결 차량 확인
  -> 차량 상세 진입 또는 Workspace 이동
  -> 기사 선택
  -> 연결 저장
  -> 상태 정상화
```

**처리 규칙:**
- Owner만 수정 가능
- Staff는 조회만 가능
- 미연결 차량은 KPI 및 필터에서 별도 노출

### 9.2 오프라인 차량 대응 프로세스

**흐름:**
```
Home/Alerts
  -> 오프라인 차량 선택
  -> Vehicle Detail 진입
  -> 최근 수신 시각 확인
  -> 장치 상태 및 최근 운행 확인
  -> 필요 시 내부 운영 지원 요청
```

**처리 규칙:**
- 모바일에서는 상태 조회 중심
- 장치 개통/재인증/강제 조치는 Internal Admin Web 영역

### 9.3 경고 차량 대응 프로세스

**흐름:**
```
Alerts
  -> 경고 알림 선택
  -> Alert Detail 확인
  -> 관련 차량 상세 이동
  -> 최근 이벤트/운행 정보 확인
  -> 필요 시 Owner가 담당자에게 전달 또는 내부 문의
```

---

## 10. 권한별 프로세스 매트릭스

| 프로세스 | Owner | Staff | Internal Admin |
|----------|-------|-------|----------------|
| 로그인 및 Home 조회 | O | O | 별도 시스템 |
| 차량 상태 조회 | O | O | O |
| 운행 이력 조회 | O | O | O |
| 알림 조회 | O | O | O |
| 담당자 생성/비활성화 | O | X | 지원용 |
| 그룹 생성/수정 | O | X | 지원용 |
| 기사 등록/수정 | O | X | 지원용 |
| 차량-기사 연결 변경 | O | X | 지원용 |
| 입·출차 위치 설정 | O | 제한적(개인수신만) | 지원용 |
| 장치 개통/인증/검증 | X | X | O |
| 내부 강제 보정 | X | X | O |

---

## 11. 메뉴 설계 원칙

### 11.1 단순성

- 하단 탭 개수는 최소화한다.
- 빈도가 낮은 기능은 Workspace 내부로 묶는다.

### 11.2 역할 분리

- 내부 운영 메뉴와 고객용 운영 메뉴를 명확히 분리한다.
- Owner는 자기 조직 범위만 관리한다.
- Staff는 운영 행위보다 조회/알림 대응에 집중한다.

### 11.3 확장성

향후 아래 기능은 확장 가능하다:
- 지도 기반 관제 강화
- 일자별 배차/교대 이력
- 다중 기사/보조 기사 연결
- 전송 결과 상세 리포트
- 조직 단위 대시보드 고도화

---

## 12. 변경 이력 (Changelog)

- **v1.0 (2026-03-25):**
  - 기존 5개 문서를 통합하여 3개 문서로 재구성
  - 프로세스 정의서(process_flow_v1) + IA/메뉴 구조도(ia_menu_v1) 통합
  - 문서 명명 규칙(TrackCar Mobile App *.md) 적용
  - 메타데이터(작성일, 버전, 상태) 추가
  - Backend Ingest 파이프라인 연동 포인트 명시
  - eTAS 전송용 필드(driver_code, vehicle_type) 의존성 추가

---

## 13. 참고 문서

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
