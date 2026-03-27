# TrackCar 인터페이스 확장 설계

- 버전: v2.1
- 수정일: 2026-03-11
- 목적: 다중 DTG 제조사 연동을 표준화하고, 동일 확장 프레임워크에서 파트너사 제공 API/배치를 수용하는 구조를 정의한다.
- 운영 기준: **DTG inbound는 MQTT only**, 파트너 outbound는 REST/배치/파일 확장 허용

---

## 1) 배경과 문제정의

TrackCar는 여러 제조사의 DTG 디바이스를 연동해야 한다. 제조사별 payload/필드/전송 제약이 다르기 때문에, 디바이스마다 비즈니스 로직을 따로 구현하면 운영 복잡도와 유지보수 비용이 급격히 증가한다.

또한 파트너사(관제/정산/물류 시스템)에게도 TrackCar 데이터를 제공해야 하므로, **디바이스 수집 표준화**와 **외부 제공 확장성**을 분리해서 설계해야 한다.

---

## 2) 설계 목표

1. **다중 제조사 수용**: DTG 제조사 추가 시 코어 로직 변경 최소화
2. **표준 이벤트 정규화**: 제조사별 입력을 TrackCar Canonical Event로 통일
3. **경계 명확화**: Inbound(DTG 수집)와 Outbound(파트너 제공)를 서로 다른 확장 정책으로 관리
4. **운영 독립성**: 인터페이스 버전별 롤아웃/롤백/비활성화 가능
5. **추적 가능성**: 인터페이스별 변환/전송 이력과 오류 원인 감사 가능

---

## 3) 전체 아키텍처(개념)

1. **Inbound Device Adapter Layer (DTG 제조사별)**
   - 수집 진입점은 **AWS IoT Core MQTT/mTLS** 로 고정
   - 제조사별 payload parser / schema validator / canonical transformer 제공
   - 디바이스 transport 다양화(REST/Batch)는 현재 표준에서 허용하지 않음
2. **Canonical Transformer**
   - 제조사 payload → TrackCar Canonical Event
3. **Business Services**
   - 운행 생성 / 이벤트 판정 / 알림 / 저장 / 외부전송 준비
4. **Outbound Adapter Layer (파트너별)**
   - Canonical/Event/집계 데이터를 파트너 포맷으로 변환
   - REST API push/pull, 배치, 파일 기반 연계를 수용

핵심 원칙: **비즈니스 코어는 제조사/파트너 포맷을 모른다.**

---

## 4) 확장 경계 정의

### 4.1 Inbound (DTG 수집)
- 표준 transport: MQTT only
- 인증: X.509 mTLS
- 온보딩: Vendor CA 기반 신뢰 + **JITR**
- 라우팅 기준: `deviceId`
- 서버 검증: device 등록, binding, interface version, subscription, dedup
- 활성화 정책: Lambda + 사업자 DB 검증 전에는 인증서를 자동 활성화하지 않음

### 4.2 Outbound (파트너/대외 제공)
- 허용 transport: REST API / batch / file
- 라우팅 기준: outbound profile / trigger policy / interface catalog
- 재시도 및 DLQ는 outbound 별도 정책 적용

즉, **Inbound와 Outbound를 동일한 “인터페이스 확장”으로 묶되 transport 정책은 분리**한다.

---

## 5) 데이터 모델 연계

### 5.1 Inbound 확장
- `dtg_interface_catalog`
- `vehicle_device_binding`
- `ingest_dedup_guard`

### 5.2 Outbound 확장
- `outbound_profile`
- `outbound_interface_catalog`
- `outbound_transmission`

### 5.3 운영/감사
- `audit_log`
- `etas_transmission_log`

---

## 6) 운영 원칙

### 6.1 Inbound 온보딩 원칙
1. 제조사 CA는 AWS IoT Core 신뢰 CA로 사전 등록한다.
2. 디바이스 최초 연결은 JITR로 등록하되 인증서 상태는 검증 전 활성화하지 않는다.
3. `deviceId`는 런타임 주 식별자이며, 인증서 subject/serial은 보조 검증 키로 사용한다.
4. vehicle 매핑은 인증서 subject에 고정하지 않고 서버 DB에서 관리한다.


1. DTG 단말 신규 연동은 MQTT/mTLS 기준으로만 승인한다.
2. 제조사별 차이는 payload mapping 계층에서 흡수한다.
3. Canonical Event 이후 비즈니스 서비스는 transport를 의식하지 않는다.
4. 파트너 제공 인터페이스는 Outbound Adapter에서 독립적으로 확장한다.
5. 문서/검증/테스트는 Inbound와 Outbound를 분리하여 관리한다.

---

## 7) 변경 이력

- v2.1 (2026-03-11)
  - DTG inbound를 MQTT only로 명시
  - Inbound 확장과 Outbound 확장을 transport 정책 기준으로 분리 정리
- v2.0 (2026-03-01)
  - 문서 목적을 “다중 DTG 확장 및 파트너 API 확장”으로 재정의

#업무/프로젝트/trackcar/산출물
