# TrackCar Installer Admin 프로세스 및 IA 설계서

- 작성일: 2026-03-25
- 버전: v1.0
- 상태: 초안

---

## 1. 문서 개요

### 1.1 목적

본 문서는 Installer Admin Web의 핵심 업무 프로세스와 정보 구조(IA)를 정의한다. Installer Admin Web은 DTG 설치 이후 실제 서비스 운영이 가능한 상태까지 개통을 완성하는 **설치 온보딩 실행 시스템**으로 정의한다.

### 1.2 적용 범위

- 대시보드, Owner/Member/Driver/Group 관리
- DTG 기기 관리, 차량 관리, 기사 관리
- 매핑 관리, Driver-Account Linking
- App Activation, 연동 검증

### 1.3 주요 보완사항

- `owner / member / driver` 계정 모델 분리
- `그룹(Group)`의 마스터를 Installer Admin 기준으로 정의
- `설치 완료`와 `서비스 활성화 완료`를 별도 상태로 분리
- `앱 초대 → 최초 로그인 → 앱 활성화 확인` 절차 추가
- 향후 `복수 기사 / 교대 운전 / 복수 모바일 단말` 확장을 고려한 흐름 반영

---

## 2. 프로세스 설계 원칙

### 2.1 기본 원칙

1. 설치기사는 현장에서 최소 단계로 등록·매핑·검증을 완료할 수 있어야 한다.
2. DTG, 차량, 기사, 그룹, 사용자 계정은 독립 등록 후 연결 가능한 구조여야 한다.
3. 설치 완료와 서비스 활성화 완료는 구분하여 추적해야 한다.
4. 모바일 앱은 운영 시스템이고, Installer Admin은 초기 개통 시스템이다.
5. 데이터 정합성 오류는 Trip 파일/전송/관제 오류로 이어질 수 있으므로 등록 단계에서 선제 검증해야 한다.
6. 현재 화면은 MVP 수준으로 단순화하되, 데이터 모델과 상태 흐름은 향후 교대 운전/다중 사용자 확장을 수용할 수 있어야 한다.

### 2.2 역할 분리 원칙

#### Installer Admin Web

- owner/member/driver 기준정보 생성
- group 마스터 생성 및 기본 구조 설정
- DTG/차량/기사 등록
- DTG-차량-기사-계정 연결
- 연동 검증
- 앱 초대 및 활성화 상태 확인

#### 모바일 앱

- 실제 사용자 로그인
- 실시간 운행정보 조회
- 차량 알림 확인
- 사용자 편의 설정
- 계정 범위 내 그룹/차량/기사 운영성 기능 수행

---

## 3. 상태 모델 정의

### 3.1 개통 단계 상태

| 상태 코드 | 의미 | 판정 주체 |
|---|---|---|
| OWNER_CREATED | Owner 생성 완료 | Admin |
| GROUP_READY | 기본 그룹 준비 완료 | Admin |
| DEVICE_REGISTERED | DTG 기기 등록 완료 | Admin |
| DEVICE_PENDING_ACTIVATION | DTG JITR 등록, 서버 검증 대기 | System |
| DEVICE_ACTIVATED | DTG 서버 검증 통과, 데이터 수신 가능 | System |
| VEHICLE_REGISTERED | 차량 등록 완료 | Admin |
| DRIVER_REGISTERED | 기사 등록 완료 | Admin |
| DEVICE_VEHICLE_MAPPED | DTG-차량 연결 완료 | Admin |
| VEHICLE_DRIVER_MAPPED | 차량-기사 연결 완료 | Admin |
| DRIVER_ACCOUNT_LINKED | Driver와 앱 계정 연결 완료 | Admin |
| INSTALLATION_VERIFIED | 서버 수신/연동 검증 완료 | Admin |
| APP_INVITED | 앱 초대 발송 완료 | Admin |
| APP_FIRST_LOGIN_DONE | 앱 최초 로그인 완료 | Mobile/App |
| SERVICE_ACTIVATED | 서비스 활성화 완료 | Admin + Mobile |

### 3.2 DTG 장치 상태

| 상태 코드 | 의미 | 비고 |
|---|---|---|
| PENDING_ACTIVATION | JITR 후 인증서 자동 등록, 서버 검증 대기 중 | 최초 연결 시 |
| REGISTERED | DTG 등록 완료, 물리 설치 전 | - |
| ACTIVE | Lambda 검증 통과, ingest 수신 가능 | Backend 기준 |
| MAPPED | 차량과 매핑됨 | - |
| ONLINE | 최근 데이터 수신 중 | - |
| OFFLINE | 장기 미수신 | - |
| ERROR | 장치 이상 상태 | - |

### 3.2 완료 상태 정의

#### 설치 완료(Installation Completed)

다음 항목이 완료되면 성립한다.

- DTG 물리 설치 완료
- DTG 등록 완료
- 차량 등록 완료
- 기사 등록 완료
- DTG-차량 매핑 완료
- 차량-기사 매핑 완료
- 서버 수신/기본 검증 완료

#### 서비스 활성화 완료(Service Activated)

다음 항목이 추가로 완료되면 성립한다.

- 앱 초대 완료
- 앱 첫 로그인 완료
- 사용자 권한/소속 자산 정상 확인
- 앱에서 정상 상태 확인 완료

즉, **설치 완료 = 현장 개통 완료**, **서비스 활성화 완료 = 실제 운영 시작 가능 상태**로 구분한다.

### 3.3 데이터 흐름과 Backend 연동

본 시스템은 Backend의 Ingest 파이프라인과 긴밀히 연동된다:

```
DTG 단말 (MQTT/mTLS)
  └── AWS IoT Core
      └── Kinesis Data Streams
          └── Lambda ingest-auth-validate
              └── 검증: deviceId, tenant, binding, interface, subscription
                  ├── PASS → Canonical Event 발행
                  └── FAIL → reject/error log
```

**핵심 연동 포인트:**
- DTG는 MQTT topic `trackcar/{deviceId}/telemetry`로 데이터 전송
- BatchSize 100 이상으로 배치 처리
- Payload 검증 시 `ignition` 필드를 차량 상태 기준으로 사용
- DynamoDB `ingest_dedup_guard`로 deviceId + batchId 멱등성 보장
- 검증 통과 후 Aurora에 Trip/Event 데이터 적재

### 3.4 eTAS 전송과 Admin 기준정보

Backend의 eTAS 전송 파이프라인은 Admin Web에서 관리하는 기준정보를 필수로 참조한다:

| Backend 필요 데이터 | Admin 관리 필드 |
|---|---|
| 자동차등록번호 | 차량번호 |
| 차대번호(VIN) | VIN |
| 자동차유형코드 | 차량유형코드 |
| 운송사업자등록번호 | Owner.운송사업자등록번호 |
| 운전자코드 | 기사.운전자코드 |

따라서 Admin Web의 차량/기사/Owner 등록 시 위 필드의 정확성이 eTAS 전송 성공에 직결된다.

---

## 4. End-to-End 서비스 플로우

### 4.1 전체 프로세스

```
[1] Owner 조회 또는 생성
    └──
[2] 기본 Group 준비
    └──
[3] DTG 기기 등록
    └──
[4] 차량 등록
    └──
[5] 기사 등록
    └──
[6] Member/Driver 계정 준비
    └──
[7] DTG-차량 매핑
    └──
[8] 차량-기사 매핑
    └──
[9] Driver-Account Linking
    └──
[10] 연동 검증 실행
    └──
[11] 설치 완료 처리
    └──
[12] 앱 초대 발송
    └──
[13] 모바일 앱 최초 로그인 확인
    └──
[14] 서비스 활성화 완료
```

### 4.2 단계별 설명

#### 1) Owner 조회 또는 생성

- 기존 고객 여부를 먼저 확인한다.
- 없으면 신규 owner를 생성한다.
- 개인/법인 유형을 구분한다.

#### 2) 기본 Group 준비

- owner 기준 기본 그룹을 자동 생성하거나 선택한다.
- 향후 차량/기사/사용자/기기의 소속 기준으로 활용한다.

#### 3) DTG 기기 등록

- 시리얼번호, 형식승인번호, 제품일련번호, 모델명 등을 등록한다.
- 아직 차량과 매핑하지 않아도 우선 등록 가능해야 한다.

#### 4) 차량 등록

- 차량번호, VIN, 차종, owner, group을 등록한다.

#### 5) 기사 등록

- 기사명, 휴대폰번호, 기사코드, 소속 owner/group을 등록한다.

#### 6) Member/Driver 계정 준비

- 앱 로그인 주체가 필요한 경우 member/driver 계정을 준비한다.
- 드라이버 엔터티와 계정은 분리 가능하지만 링크 구조를 갖는다.

#### 7) DTG-차량 매핑

- 설치된 DTG를 차량에 연결한다.

#### 8) 차량-기사 매핑

- 차량의 기본 기사 또는 사용 기사를 연결한다.
- MVP에서는 기본 기사 1인 기준으로 처리하되, 데이터 구조는 복수 기사 확장을 허용한다.

#### 9) Driver-Account Linking

- 실제 앱 로그인 계정과 driver 엔터티를 연결한다.

#### 10) 연동 검증 실행

- 서버 수신 여부, GPS, heartbeat, 데이터 반영 여부를 확인한다.

#### 11) 설치 완료 처리

- 설치기사 관점의 개통 완료 상태를 기록한다.

#### 12) 앱 초대 발송

- owner/member/driver 대상 앱 초대 링크 또는 임시 인증정보를 발송한다.

#### 13) 모바일 앱 최초 로그인 확인

- 초대 수락 후 앱 첫 로그인 및 자산 표시 여부를 확인한다.

#### 14) 서비스 활성화 완료

- 실제 사용자 사용 가능 상태를 최종 완료로 기록한다.

---

## 5. 세부 프로세스 정의

### 5.1 Owner 조회/생성 프로세스

#### 목적

- 기존 고객 재사용
- 중복 owner 생성 방지
- 개인/법인 유형 식별

#### 입력

- 휴대폰번호
- 사업자번호
- 차량번호
- 회사명 또는 이름

#### 처리 규칙

1. 휴대폰번호/사업자번호/차량번호 기준으로 기존 owner를 조회한다.
2. 기존 owner가 있으면 해당 owner를 선택한다.
3. 기존 owner가 없으면 신규 생성 화면으로 진입한다.
4. 개인 owner는 기본 그룹 1개를 자동 생성할 수 있다.
5. 법인 owner는 기본 그룹 생성 후 추가 그룹 확장을 허용한다.

#### 결과

- `OWNER_CREATED`
- `GROUP_READY`

---

### 5.2 Group 준비 프로세스

#### 목적

- 차량/기사/사용자/기기의 공통 소속 기준 마련
- 모바일 앱과 동일한 권한/조회 범위 기준 제공

#### 처리 규칙

1. 신규 owner 생성 시 기본 그룹을 자동 생성할 수 있다.
2. 차량 등록 시 group 선택을 요구한다.
3. 기사 등록 시 group 선택을 요구한다.
4. member/driver 계정은 최소 1개 group에 소속되어야 한다.
5. group 마스터 변경은 Admin에서만 수행한다.

---

### 5.3 DTG 기기 등록 프로세스

#### 목적

- 설치 기기의 식별성과 추적성 확보
- 표준사양 기반 기초정보 등록

#### 입력

- 시리얼번호
- 형식승인번호
- 제품일련번호
- 모델명
- 펌웨어 버전
- 통신정보(optional)
- 설치 메모(optional)

#### 처리 규칙

1. 시리얼번호와 제품일련번호는 중복 허용하지 않는다.
2. DTG는 등록만 먼저 하고 추후 차량과 매핑할 수 있다.
3. 등록 직후 기본 상태는 `DEVICE_REGISTERED` 이다.
4. 마지막 수신시각이 없으면 아직 온라인으로 보지 않는다.

#### 결과

- `DEVICE_REGISTERED`

---

### 5.4 차량 등록 프로세스

#### 목적

- DTG/기사/앱과 연결될 차량 기준정보 생성

#### 입력

- 차량번호
- VIN
- 차종
- owner
- group
- 상태

#### 처리 규칙

1. 차량번호 중복 여부를 확인한다.
2. 동일 owner 내 중복 등록을 방지한다.
3. DTG 없이 차량만 선등록 가능하다.
4. 기본 group 미지정 시 owner 기본 group을 사용한다.

#### 결과

- `VEHICLE_REGISTERED`

---

### 5.5 기사 등록 프로세스

#### 목적

- 차량 운행 주체와 전송/분석 데이터의 운전자 식별정보 생성

#### 입력

- 기사명
- 휴대폰번호
- 기사코드/사번
- owner
- group
- 상태

#### 처리 규칙

1. 기사 엔터티와 로그인 계정은 분리 가능하다.
2. 휴대폰번호 또는 내부 코드 중 최소 1개 식별수단을 확보한다.
3. 기사 등록 직후 앱 계정이 없어도 된다.
4. 향후 복수 차량/교대 운전에 대응할 수 있도록 다건 매핑 가능 구조를 전제로 한다.

#### 결과

- `DRIVER_REGISTERED`

---

### 5.6 Member/Driver 계정 준비 프로세스

#### 목적

- 실제 앱 로그인 사용자 확보
- owner/member/driver 권한 분리

#### 처리 규칙

1. owner는 대표 계정이다.
2. member는 운영/관리 역할의 사용자 계정이다.
3. driver는 실제 운전자용 사용자이며, driver 엔터티와 계정 링크가 필요할 수 있다.
4. 설치 현장에서는 계정을 즉시 만들거나 초대 대기 상태로 둘 수 있다.
5. 계정 생성만으로 서비스 활성화 완료로 보지 않는다.

---

### 5.7 DTG-차량 매핑 프로세스

#### 목적

- 설치된 물리 기기와 차량의 실제 연결관계 정의

#### 입력

- device_id
- vehicle_id
- 적용 시작일시
- 활성 여부

#### 처리 규칙

1. DTG는 동시에 1대 차량에만 활성 연결한다.
2. 차량도 동시에 1개 활성 DTG만 허용한다.
3. 재설치/교체를 고려하여 이력형 구조로 관리한다.
4. 매핑 후 데이터 수신은 해당 차량 귀속으로 처리한다.

#### 결과

- `DEVICE_VEHICLE_MAPPED`

---

### 5.8 차량-기사 매핑 프로세스

#### 목적

- 차량 운행 주체 지정
- 향후 운행기록/알림/관제의 기본 운전자 설정

#### 입력

- vehicle_id
- driver_id
- primary 여부
- 적용 시작일시

#### 처리 규칙

1. MVP에서는 차량당 기본 기사 1명을 우선 관리한다.
2. 단, 데이터 구조는 다건 매핑을 허용한다.
3. 교대 운전 확장을 위해 이력 구조를 유지한다.
4. 기사 미매핑 상태에서도 설치 검증은 가능하지만 서비스 활성화는 제한될 수 있다.

#### 결과

- `VEHICLE_DRIVER_MAPPED`

---

### 5.9 Driver-Account Linking 프로세스

#### 목적

- 기사 엔터티와 실제 앱 로그인 계정 연결

#### 입력

- driver_id
- user_account_id
- link_type
- linked_at

#### 처리 규칙

1. driver와 account는 1:1 기본 연결을 우선 지원한다.
2. 향후 복수 단말/교대 운전 대응을 위해 별도 링크 테이블을 사용한다.
3. 계정 미연결 기사도 등록 가능하지만 앱 활성화는 불가하다.

#### 결과

- `DRIVER_ACCOUNT_LINKED`

---

### 5.10 연동 검증 프로세스

#### 목적

- 설치 직후 서비스 운용 가능성 확인

#### 검증 항목

- 기기 등록 여부
- DTG-차량 매핑 여부
- 차량-기사 매핑 여부
- 최근 N분 이내 heartbeat 수신 여부
- GPS 수신 여부
- 속도/거리 등 운행 데이터 수신 여부
- 서버 반영 여부
- 앱 계정/초대 상태 여부

#### 판정 규칙

##### PASS

- 기기 등록 정상
- 매핑 정상
- 최근 데이터 수신 정상
- 앱 준비 또는 앱 로그인 확인 완료

##### WARNING

- 수신은 있으나 일부 연결 정보 누락
- 기사 미매핑
- 계정 미링크
- 앱 초대 미완료

##### FAIL

- 기기 미등록
- DTG-차량 매핑 없음
- 최근 수신 없음
- GPS/heartbeat 모두 미확인

#### 결과

- `INSTALLATION_VERIFIED`
- 검증 리포트 저장

---

### 5.11 설치 완료 처리 프로세스

#### 목적

- 설치기사 기준의 현장 개통 완료 처리

#### 완료 조건

- owner/group 준비 완료
- DTG 등록 완료
- 차량 등록 완료
- 기사 등록 완료
- DTG-차량 매핑 완료
- 차량-기사 매핑 완료
- 연동 검증 결과가 `PASS` 또는 허용 가능한 `WARNING`

#### 처리 규칙

1. 설치 완료는 현장 설치 종료의 의미다.
2. 설치 완료만으로 서비스 활성화 완료로 보지 않는다.
3. 모바일 앱 로그인 전까지는 후속 활성화 상태를 계속 추적한다.

---

### 5.12 앱 초대 및 최초 로그인 프로세스

#### 목적

- 등록된 사용자에게 실제 서비스 접근권한 제공

#### 처리 방식

- SMS 초대 링크
- 임시 비밀번호 발급
- OTP/초대코드 방식

#### 흐름

1. Admin이 owner/member/driver에게 앱 초대를 발송한다.
2. 사용자가 앱 설치 후 로그인한다.
3. 최초 로그인 후 비밀번호 변경/동의 절차를 수행한다.
4. 소속 그룹, 연결 차량, 연결 기사/기기 정보를 앱에서 확인한다.
5. Admin 또는 시스템이 앱 첫 로그인 완료를 확인한다.

#### 결과

- `APP_INVITED`
- `APP_FIRST_LOGIN_DONE`

---

### 5.13 서비스 활성화 완료 프로세스

#### 목적

- 설치 완료 이후 실제 운영 시작 가능 상태를 최종 확정

#### 완료 조건

- 앱 초대 완료
- 앱 첫 로그인 완료
- 사용자 권한/소속 자산 정상 확인
- 앱 상태 화면에서 정상 연결 여부 확인

#### 처리 규칙

1. 서비스 활성화 완료는 모바일 앱 기준 최종 개통 상태다.
2. 설치 완료와 별도 이력으로 관리한다.
3. 앱 첫 로그인 후 자산 노출이 비정상일 경우 활성화 완료로 처리하지 않는다.

#### 결과

- `SERVICE_ACTIVATED`

---

## 6. 예외 및 재처리 플로우

### 6.1 기존 고객 재설치

```
기존 Owner 조회
  └── 기존 차량 조회
      └── DTG 교체 또는 재매핑
          └── 검증 재실행
              └── 상태 갱신
```

### 6.2 기사만 미등록인 경우

```
DTG 등록 완료
  └── 차량 등록 완료
      └── 기사 미등록
          └── WARNING 상태 검증
              └── 기사 등록 후 재검증
```

### 6.3 앱 로그인 실패

```
설치 완료
  └── 앱 초대 발송
      └── 최초 로그인 실패
          └── 임시 비밀번호 재발급/초대 재전송
              └── 앱 로그인 재확인
                  └── 서비스 활성화 완료
```

### 6.4 수신 불량 또는 오프라인

```
DTG 등록/매핑 완료
  └── heartbeat/GPS 미수신
      └── FAIL 판정
          └── 설치 상태/통신 상태 점검
              └── 재검증 실행
```

---

## 7. IA 설계 원칙

### 7.1 역할 분리 원칙

Installer Admin Web은 **설치 / 등록 / 매핑 / 연동 검증 / 초기 활성화 관리**를 담당한다.

모바일 앱은 **운행 / 관제 / 알림 / 사용자 설정 / 현장 사용**을 담당한다.

따라서 Installer Admin Web의 IA는 다음 관점으로 구성한다.

- 설치 시점의 기준정보 생성
- 장치·차량·기사·계정 연결 관리
- 연동 검증 및 활성화 상태 관리
- 모바일 앱으로 인계할 초기 상태 확정

### 7.2 마스터 데이터 원칙

MVP 기준 마스터 주체는 아래와 같이 정의한다.

- Owner / Member / Driver 계정 마스터: Installer Admin
- Group 마스터: Installer Admin
- DTG Device 마스터: Installer Admin
- Vehicle / Driver 기준정보 마스터: Installer Admin
- 모바일 사용자 편의설정 마스터: Mobile App

### 7.3 단계형 구조 원칙

사용 흐름이 복잡하지 않도록 다음 순서를 기준으로 메뉴를 설계한다.

1. 고객/계정 생성
2. 그룹 생성 및 소속 정리
3. DTG 등록
4. 차량 등록
5. 기사 등록
6. 매핑
7. 연동 검증
8. 앱 활성화 확인

---

## 8. 최상위 메뉴 구조

```text
Installer Admin Web
├─ Dashboard
├─ Accounts
│  ├─ Owner Accounts
│  ├─ Member Accounts
│  └─ Driver Accounts
├─ Groups
├─ DTG Devices
├─ Vehicles
├─ Drivers
├─ Mappings
│  ├─ Device-Vehicle Mapping
│  ├─ Vehicle-Driver Mapping
│  ├─ Driver-Account Linking
│  └─ Quick Mapping
├─ Verification
├─ App Activation
└─ System
```

---

## 9. 메뉴 구조 상세

### 9.1 Dashboard

설치기사/운영자 관점에서 전체 설치 자산과 연동 상태를 한눈에 확인하는 메뉴다.

#### 하위 영역

- 설치 차량 현황 요약
- DTG 연동 상태 차트
- 차량-기사 매핑 상태 차트
- 검증 상태 차트
- 앱 활성화 상태 차트
- 최근 이상 대상 목록

#### 주요 표시 정보

- 총 설치 차량 수, 활성 DTG 수, 미연결 DTG 수
- 차량-기사 매핑 완료 수
- 검증 PASS / WARNING / FAIL 수
- 앱 초대 완료 수, 앱 최초 로그인 완료 수, 서비스 활성화 완료 수

---

### 9.2 Accounts

설치 시 생성되거나 연결되는 계정 체계를 관리하는 메뉴다. `Owner / Member / Driver`로 분리한다.

#### Owner Accounts

- 고객의 대표 계정 또는 사업주 계정을 관리한다.
- 개인 / 법인 구분, 휴대폰번호 / 사업자번호 중복 확인, 기본 그룹 자동 생성, 앱 초대 대상 지정

#### Member Accounts

- 회사 관리자, 차량 담당자, 배차 담당자 등 운영용 사용자 계정을 관리한다.
- Owner 소속 지정, Group 소속 지정, 역할 지정, 앱 또는 웹 사용 권한 지정

#### Driver Accounts

- 실제 운전자가 모바일 앱을 사용하는 경우의 로그인 계정을 관리한다.
- Driver 엔터티와 계정 연결, 앱 초대 발송, 최초 로그인 상태 확인, 활성화 상태 확인

---

### 9.3 Groups

Group은 차량/기사/계정 소속의 기준 단위이며, MVP에서는 Installer Admin이 마스터로 관리한다.

#### 주요 기능

- Owner별 기본 그룹 생성
- 차량 / Driver / Member 소속 지정
- 비활성 그룹 관리

#### 설계 원칙

- 개인 owner 생성 시 기본 그룹 자동 생성
- 법인 owner는 복수 그룹 구성이 가능
- 모바일 앱은 그룹 조회 및 권한 범위 내 일부 편집만 수행

---

### 9.4 DTG Devices

설치된 DTG 장치의 등록, 식별, 연결, 상태를 관리하는 메뉴다.

#### 주요 기능

- 시리얼번호, 형식승인번호, 제품일련번호 등록
- 모델 / 펌웨어 관리
- 최근 수신 시각 확인
- 차량 연결 상태 확인
- 연동 상태 확인

#### 비고

장치의 식별값과 설치 상태는 Installer Admin이 마스터로 관리한다.

---

### 9.5 Vehicles

설치 대상 차량의 기준정보와 연결 상태를 관리한다.

#### 주요 기능

- 차량번호 / VIN 등록
- Owner / Group 지정
- 연결 DTG 조회
- 기본 Driver 조회
- 활성 상태 관리

---

### 9.6 Drivers

실제 운전자 기준정보를 관리한다. 계정(Account)과 분리된 마스터 정보로 관리하며, 필요 시 Driver Account와 연결한다.

#### 주요 기능

- 기사명 / 휴대폰번호 / 기사코드 등록
- Owner / Group 지정
- 기본 차량 확인
- 계정 연결 여부 확인

#### 확장 고려

MVP UI는 단순하게 유지하되, 데이터 구조는 복수 차량 / 복수 단말 / 교대 운전 확장 가능하도록 설계한다.

---

### 9.7 Mappings

설치 시점의 핵심 연결 관계를 관리하는 메뉴다.

#### 하위 메뉴

- Device-Vehicle Mapping: DTG와 차량의 연결 관리, 1개 DTG의 현재 활성 차량 확인, 차량 교체/재설치 대비 이력 관리
- Vehicle-Driver Mapping: 차량과 기사 연결 관리, 기본 기사 지정, 보조/추가 기사 확장 가능 구조 고려
- Driver-Account Linking: Driver 엔터티와 Driver Account 연결, 모바일 앱 로그인 주체 연결
- Quick Mapping: 현장 즉시 처리를 위한 핵심 메뉴

---

### 9.8 Verification

설치 후 서버 연동과 데이터 수신 상태를 검증하는 메뉴다.

#### 주요 기능

- DTG 등록 여부 확인
- DTG-차량 매핑 여부 확인
- 차량-기사 매핑 여부 확인
- 최근 heartbeat 수신 확인
- GPS 수신 여부 확인
- PASS / WARNING / FAIL 판정

#### 비고

이 메뉴는 **설치 완료 판정**의 핵심 기준이 된다.

---

### 9.9 App Activation

설치 완료 이후, 실제 모바일 앱 사용자 계정이 정상적으로 활성화되었는지 관리하는 메뉴다.

#### 주요 기능

- 앱 초대 발송 상태 확인
- 최초 로그인 완료 여부 확인
- 비밀번호 초기화 / 재초대
- 서비스 활성화 완료 처리

#### 상태 정의

- INVITATION_PENDING
- INVITED
- FIRST_LOGIN_DONE
- ACTIVATED
- ACTIVATION_FAILED

---

### 9.10 System

MVP 운영에 필요한 최소한의 시스템 관리 메뉴다.

#### 하위 메뉴

- Users: 설치기사 / 운영자 계정 관리
- Roles: 역할별 권한 관리
- Common Codes: 공통 상태코드 관리
- Audit Log: 주요 변경이력 확인

---

## 10. 사용자 역할별 메뉴 접근 구조

### 10.1 Installer

#### 접근 메뉴

- Dashboard
- Accounts (Owner/Member/Driver)
- Groups
- DTG Devices
- Vehicles
- Drivers
- Mappings
- Verification
- App Activation(조회/재초대 일부)

#### 특징

현장 설치 / 등록 / 매핑 / 검증 중심 권한

### 10.2 Operator

#### 접근 메뉴

- Dashboard
- Accounts (Owner/Member/Driver)
- Groups
- DTG Devices
- Vehicles
- Drivers
- Mappings
- Verification
- App Activation

#### 특징

재검증, 활성화 지원, 오류 대응 중심 권한

### 10.3 Admin

#### 접근 메뉴

- 전 메뉴 (계정/권한/코드/감사 로그 포함)

#### 특징

전체 관리 권한

---

## 11. 정보 구조 관점의 핵심 연결 관계

```text
Owner
 ├─ Groups
 │   ├─ Vehicles
 │   │   └─ Device Mapping
 │   ├─ Drivers
 │   │   └─ Driver Account Linking
 │   └─ Members
 └─ App Activation Status
```

### 설명

- Owner는 최상위 고객 단위다.
- Group은 소속/관리 단위다.
- Vehicle과 Driver는 Group 하위 자산이다.
- Device는 Vehicle에 연결된다.
- Driver는 Driver Account와 연결되어 모바일 앱 사용 주체가 된다.
- 앱 활성화 상태는 계정/소속/자산 연결 상태와 함께 관리된다.

---

## 12. 화면 이동 핵심 플로우 기준 메뉴 구조

### 12.1 신규 설치 / 최초 개통 플로우

```
Accounts > Owner Accounts
Groups
DTG Devices
Vehicles
Drivers
Mappings > Quick Mapping
Verification
App Activation
```

### 12.2 기존 고객 장치 추가 설치 플로우

```
Accounts > Owner Accounts 조회
Groups 조회
DTG Devices 등록
Vehicles 조회 또는 등록
Drivers 조회 또는 등록
Mappings
Verification
App Activation
```

### 12.3 기사 앱 활성화 지원 플로우

```
Drivers
Accounts > Driver Accounts
App Activation
```

---

## 13. 모바일 앱과의 IA 경계 정의

### Installer Admin이 담당하는 영역

- 기준정보 생성
- 마스터 ID 발급
- 소속 그룹 관리
- DTG 식별정보 등록
- 차량/기사 연결
- 설치 검증
- 앱 초대 및 활성화 상태 관리

### Mobile App이 담당하는 영역

- 로그인 및 실제 사용
- 운행/관제/알림 확인
- 사용자 편의설정
- 현장 운전자 사용 흐름
- 알림 확인 및 상태 모니터링

### 경계 원칙

- Admin은 **개통 전/직후 초기화** 중심
- Mobile App은 **운영 중 사용 경험** 중심

---

## 14. 대시보드 반영 지표 연결

프로세스 결과는 대시보드의 차트/KPI로 연결된다.

### KPI 예시

- 총 설치 차량 수
- 활성 DTG 수
- 설치 완료 수
- 서비스 활성화 완료 수
- 기사 미매핑 차량 수
- 앱 미활성 계정 수
- 최근 24시간 수신 이상 차량 수

### 상태 차트 예시

- DTG 상태: Registered / Mapped / Online / Offline / Error
- 차량 개통 단계: Registered / Mapped / Verified / Installed / Activated
- 앱 활성화 단계: Invited / First Login Done / Activated

---

## 15. 변경 이력 (Changelog)

- **v1.1 (2026-03-25):**
  - DTG 장치 상태에 PENDING_ACTIVATION, ACTIVE 추가
  - Backend Ingest 파이프라인 연동 포인트 명시
  - eTAS 전송을 위한 Admin 기준정보 의존성 명시
  - ignition 상태 필드 기준 설명 추가

- **v1.0 (2026-03-25):**
  - 기존 프로세스 정의서(installer_admin_process_flow_v2.md)와 IA 메뉴 구조서(installer_admin_ia_menu_v2.md) 통합
  - 문서 명명 규칙(TrackCar Installer Admin *.md) 적용

---

## 16. 참고 문서

본 문서는 다음 Backend 문서 및 참고문서를 참조하여 작성되었습니다.

### Backend 문서

| 문서명 | 버전 | 핵심 내용 |
|--------|------|----------|
| TrackCar 플랫폼 아키텍처 상세 설계 | v2.4.0 | DTG 수집 파이프라인, 저장소 계층, eTAS 전송 |
| DTG 단말기 ↔ TrackCar IoT Core 연동 요구사항 | v1.6 | MQTT/mTLS 연동, Payload 규격, ignition 상태 |
| TrackCar Ingest 검증 명세서 | v2.2 | 9단계 검증, Canonical 이벤트, ingest 파이프라인 |
| TrackCar 외부 연계 전송 처리 명세서 | v1.1 | eTAS 전송, 기준정보 보강, 멱등키 |

### 참고 문서 (규정)

| 문서명 | 관련 항목 |
|--------|----------|
| `[별표 1]` 운행기록장치의 세부기준 | DTG 기본 사양 |
| `[별표 2]` 운행기록의 배열순서 | 운전자코드, 자동차유형코드 배치 |
| `[별표 3]` 운행기록의 파일명 | eTAS 파일명 생성 규격 |
| `[별표 4]` 자동차등록번호의 표기 | 차량번호 형식 검증 |
