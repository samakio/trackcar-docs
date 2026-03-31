# DTG 단말기 ↔ TrackCar IoT Core 연동 요구사항

- 작성일: 2026-02-25
- 상태: Draft (제조사 전달 및 협의용)
- 기준 문서:
  - `TrackCar 디바이스 Telemetry 인터페이스 명세`
  - `TrackCar 디바이스 인터페이스 제조사 회신 템플릿`
  - `TrackCar Ingest 검증 명세서`
- 운영 기준: DTG 수집 경로는 **MQTT only**

---

## 1) 목적

본 문서는 TrackCar 플랫폼의 AWS IoT Core 기반 연동을 위한 디바이스(단말) 측 구현 요구사항을 정의한다.  
제조사는 본 문서를 기준으로 펌웨어 구현 가능 여부, 제약사항, 일정 및 검증 결과를 회신한다.

---

## 2) 핵심 원칙

1. **책임 분리**
   - 디바이스는 디바이스가 결정 가능한 필드만 전송한다.
   - `tenantId`, `vehicleId`, `interfaceId`, `interfaceVersion`, `outbound*` 는 서버 매핑 책임이다.

2. **식별/상태 원칙**
   - 운영상 단말 식별은 `deviceId` 기준이다.
   - 인증서 Subject 식별값(`CN`, `serialNumber`)은 단말 하드웨어 또는 인증서 고유 식별용으로 사용한다.
   - 플랫폼은 `deviceId` 와 인증서 Subject 식별값의 매핑 유효성을 사업자 DB 기준으로 검증한다.
   - 소속/구독 상태/바인딩 상태는 서버가 판정한다.

3. **전송 규칙**
   - 전송 주기: 30초 1회
   - 데이터 해상도: 1초 샘플
   - 배치당 샘플 수: 권장 30건(허용 28~32)

4. **멱등/재전송**
   - 동일 배치 재전송 시 `batchId` 및 각 샘플 `ts/seq` 는 불변이어야 한다.

5. **표준 경로**
   - 신규 단말 기준은 MQTT/mTLS 전용이며 REST ingest는 현재 표준에서 제외한다.

---

## 3) 통신/보안 요구사항

### 3.1 MQTT 연결

- 프로토콜: MQTT over TLS (MQTTS)
- 포트: `8883`
- TLS: 1.2 이상 필수
- QoS: telemetry QoS 1 권장
- MQTT `clientId` 는 원칙적으로 `deviceId` 와 동일하게 사용한다.
- 예외가 필요한 경우 사전 협의가 필요하다.
- 동일 `clientId` 의 중복 동시 연결은 운영상 허용하지 않는다.

### 3.2 인증(mTLS)

- 디바이스별 X.509 클라이언트 인증서 및 개인키 탑재 필수
- 디바이스 인증서는 **단말별 고유 인증서**를 원칙으로 하며, 공용 인증서 공유 방식은 허용하지 않는다.
- 개인키는 추출 방지 저장소(secure storage) 사용 권장
- 서버 인증서 체인 검증 필수
- 제조사 Vendor CA는 AWS IoT Core 신뢰 CA로 사전 등록한다.
- Vendor CA는 단말별 디바이스 클라이언트 인증서의 발급 주체로 사용한다.
- 디바이스 최초 연결은 **JITR** 기준으로 처리한다.
- 인증서 자동 등록은 허용하되, **Lambda + 사업자 DB 검증 전에는 자동 활성화하지 않는다.**
- 본 문서에서 인증서 Subject 규칙은 Vendor CA 인증서가 아닌 **디바이스 클라이언트 인증서**를 의미한다.
- 디바이스 인증서 Subject 권장 규칙은 다음과 같다.
  - `CN = deviceId` 또는 `deviceSerial`
  - `serialNumber = hardwareId` 또는 `IMEI`
  - `OU = vendorCode`
  - `O = TrackCar` 또는 협의된 서비스 식별자
- 제조사는 실제 적용 Subject profile(`CN`, `serialNumber` 매핑값)을 사전 회신/등록해야 한다.
- `vehicleId`, `tenantId` 는 인증서 Subject의 1차 식별자로 사용하지 않는다.
- 차량 및 테넌트 매핑은 서버 DB 기준으로 판정한다.
- IoT Core endpoint는 플랫폼팀이 환경별(dev/stg/prod)로 별도 제공한다.

### 3.3 토픽 규격

#### A. 운영 토픽

- Publish (required)
  - `dtg/{deviceId}/telemetry`
- Subscribe (optional)
  - `dtg/{deviceId}/config`

#### B. 정책 원칙

- 토픽은 `deviceId` 기준으로 단순화하며, tenant/권한 판정은 서버 매핑 기준으로 수행한다.
- telemetry publish는 필수다.
- `config` subscribe는 선택이다.
- ingest 단계에서 아래를 반드시 검증한다.
  1. `deviceId` 기준 서버 tenant/binding/interface 매핑 유효성
  2. 해당 시점 구독 상태 허용 여부
  3. `deviceId` 와 `batchId` 조합 멱등성
- 검증 실패 시 적재 거부 또는 quarantine 처리한다.
- `cmd`, `ota`, `event` 토픽은 현재 범위에서 제외하며, 후속 버전에서만 재검토한다.

### 3.4 부팅/초기화 워크플로우

1. 단말 부팅
2. mTLS로 AWS IoT Core 연결 시도
3. 최초 연결 시 JITR 등록 처리
4. Lambda가 사업자 DB 기준으로 `deviceId`, 인증서 Subject 식별값(`CN`, `serialNumber`), tenant, vehicle binding, 폐기/중복 상태를 검증
5. 검증 통과 시 플랫폼은 AWS IoT Core 측에서 인증서를 활성화하고, 필요 시 Thing 및 Policy 연결을 수행
6. 필요 시 `dtg/{deviceId}/config` subscribe
7. `dtg/{deviceId}/telemetry` publish 시작
8. 서버가 `deviceId` 기반 tenant/binding/interface 매핑을 수행

### 3.5 통신 음영 지역 예외 처리

- 통신 불가 시 로컬 버퍼링 후 재전송
- tenant 캐시 동기화는 요구하지 않는다.
- 재전송 시 원본 `batchId/seq/ts` 를 변경하지 않는다.

---

## 4) Payload 요구사항 (JSON)

디바이스 payload는 `TrackCar 디바이스 Telemetry 인터페이스 명세` 의 Device Payload v1을 따른다.

### 4.1 최상위 필드

| 필드명 | 타입 | 필수 여부 | 설명 |
|---|---|---|---|
| `schemaVer` | string | Y | Payload 스키마 버전 |
| `deviceId` | string | Y | 단말기 고유 식별자 |
| `batchId` | string | Y | 배치 전송 단위 식별자 |
| `batchStartAt` | string (ISO8601 datetime) | Y | 배치 내 첫 데이터 시각 |
| `batchEndAt` | string (ISO8601 datetime) | Y | 배치 내 마지막 데이터 시각 |
| `sentAt` | string (ISO8601 datetime) | Y | 단말이 해당 payload를 전송한 시각 |
| `points` | array<object> | Y | 위치/주행 데이터 포인트 목록 |
| `vehicleNo` | string | N | 차량번호 |
| `manufacturer` | string | N | 단말 제조사명 |
| `model` | string | N | 단말 모델명 |
| `fwVer` | string | N | 단말 펌웨어 버전 |

### 4.2 `points[]` 내부 필드

| 필드명 | 타입 | 필수 여부 | 설명 |
|---|---|---|---|
| `ts` | string (ISO8601 datetime) | Y | 개별 포인트의 측정 시각 |
| `seq` | integer | Y | 포인트 순번. 샘플 생성 순서를 나타내는 증가값 |
| `lat` | number | Y | 위도(WGS84 기준) |
| `lng` | number | Y | 경도(WGS84 기준) |
| `speedKph` | number | Y | 속도(km/h) |
| `ignition` | boolean | Y | 시동 상태(차량 가동 상태의 단일 기준 필드) |
| `heading` | number | N | 진행 방향(방위각) |
| `odoKm` | number | N | 누적 주행거리(km) |
| `rpm` | integer | N | 엔진 RPM |
| `brake` | boolean | N | 브레이크 상태 |
| `accelX` | number | N | X축 가속도 |
| `accelY` | number | N | Y축 가속도 |
| `accelZ` | number | N | Z축 가속도 |
| `missingReason` | string | N | 비정상/보정 샘플 발생 사유 코드 |

- `tenantId` 는 디바이스가 전송하지 않으며, 서버 매핑으로 결정한다.

### 4.3 문서에 바로 넣기 좋은 JSON 예시

```json
{
  "schemaVer": "v1",
  "deviceId": "DTG-001",
  "batchId": "BATCH-20260312-0001",
  "batchStartAt": "2026-03-12T09:00:00Z",
  "batchEndAt": "2026-03-12T09:04:00Z",
  "sentAt": "2026-03-12T09:04:10Z",
  "vehicleNo": "12가3456",
  "manufacturer": "ABC DTG",
  "model": "DTG-X1",
  "fwVer": "1.0.3",
  "points": [
    {
      "ts": "2026-03-12T09:00:00Z",
      "seq": 1001,
      "lat": 37.5665,
      "lng": 126.9780,
      "speedKph": 48.5,
      "ignition": true,
      "heading": 182.4,
      "odoKm": 12345.6,
      "rpm": 2100,
      "brake": false,
      "accelX": 0.01,
      "accelY": -0.03,
      "accelZ": 0.98
    }
  ]
}
```

---

## 5) 필드 제약(추가)

- `vehicleNo`: 최대 20자, 권장 문자셋(한글/영문/숫자/`-`/공백)
- `manufacturer`: 최대 40자
- `model`: 최대 40자
- 시간 필드(`ts`, `batchStartAt`, `batchEndAt`, `sentAt`): ISO8601, 오프셋 포함 권장
- 좌표: WGS84 범위 준수(`lat -90~90`, `lng -180~180`)

---

## 6) `seq` / `missingReason` 구현 필수 규칙

1. `seq`
   - 샘플 생성 시 1씩 증가
   - 같은 배치 내 오름차순
   - 재전송 시 원본 유지

2. `missingReason`
   - 정상 샘플: 생략
   - 비정상/보정 샘플: 코드 필수
   - 권장 코드: `GPS_LOST`, `SENSOR_ERROR`, `BUFFER_OVERFLOW`, `CLOCK_DRIFT_CORRECTED`

---

## 7) 오프라인/장애 대응

- 통신 단절 시 로컬 버퍼 저장
- 복구 시 과거 데이터 순차 재전송
- QoS1 PUBACK 미수신 시 재전송
- 재전송 시 중복 가능(서버 멱등 처리 전제)
- 버퍼링 정책(보존 시간/최대 건수/overflow 시 drop 규칙)은 제조사 회신 시 명시한다.

---

## 8) 운영 정책(중요)

- DTG ingest 표준 경로는 **AWS IoT Core MQTT(mTLS) 단일 경로**다.
- 신규 단말 및 현재 기준 설계 문서는 REST ingest를 가정하지 않는다.
- 과거 호환/전환 이슈가 있더라도 이는 별도 예외 운영 주제로 관리하며, 현재 제조사 구현 기준에는 포함하지 않는다.

---

## 9) 제조사 회신 필수 항목

1. X.509 mTLS 기반 MQTT 연결 가능 여부
2. 단말별 고유 인증서 발급/탑재 가능 여부 및 공용 인증서 미사용 보장 여부
3. 인증서 Subject profile(`CN`, `serialNumber`, `OU`, `O`) 적용 계획
4. MQTT `clientId = deviceId` 정책 준수 가능 여부
5. 고정 telemetry topic(`dtg/{deviceId}/telemetry`) 전송 가능 여부
6. 선택 config topic(`dtg/{deviceId}/config`) 지원 가능 여부
7. 오프라인 버퍼(시간/건수), 재전송 정책, overflow 정책
8. `seq` / `missingReason` 규칙 구현 가능 여부
9. 개발/시험/양산 일정

---

## 10) 변경 이력

- v1.6 (2026-03-18): 차량 상태 필드를 `ignition` 단일 기준으로 정리하고 `engineOn` 속성 제거
- v1.5 (2026-03-13): 디바이스 인증서 Subject 권장 규칙, 단말별 고유 인증서 원칙, `clientId = deviceId` 정책, JITR 이후 AWS 활성화/연결 처리 범위 반영
- v1.4 (2026-03-11): MQTT only 기준 확정, REST 병행 운영 가정 제거, config subscribe를 선택 항목으로 정리
- v1.3 (2026-02-25): MVP 최소 토픽(config, telemetry)로 단순화, cmd/ota/event는 범위 제외(향후 확장 Reserved)
- v1.2 (2026-02-25): mTLS, deviceId 고정 토픽, 부팅 동기화/음영지역 예외, ingest 검증 원칙 반영
- v1.0 (2026-02-25): 최초 작성

#업무/프로젝트/trackcar/산출물
