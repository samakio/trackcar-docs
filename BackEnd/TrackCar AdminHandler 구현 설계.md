# TrackCar AdminHandler 구현 설계

## 1. 설계 근거

이 문서는 아래 파일을 기준으로 작성한다.

- API Spec: `/Users/aircha/development/opencode/trackcar/docs/FrontEnd/TrackCar Installer Admin Web API Specification.md`
- Lambda 진입점: `/Users/aircha/development/opencode/trackcar/aws-functions/src/main/java/com/trackcar/api/ApiAdapterHandler.java`
- Mobile 패턴 기준: `/Users/aircha/development/opencode/trackcar/aws-functions/src/main/java/com/trackcar/api/handlers/MobileHandler.java`
- Aurora 연동: `/Users/aircha/development/opencode/trackcar/aws-functions/src/main/java/com/trackcar/common/repository/AuroraRepository.java`
- Cognito 토큰 검증: `/Users/aircha/development/opencode/trackcar/aws-functions/src/main/java/com/trackcar/common/util/CognitoAuthValidator.java`
- Cognito 사용자 관리: `/Users/aircha/development/opencode/trackcar/aws-functions/src/main/java/com/trackcar/common/util/CognitoUserManager.java`

본 문서는 설계만 다루며, Java 구현 코드는 포함하지 않는다.

---

## 2. 현재 코드 기준 관찰 사항

### 2.1 현재 Admin API 배선

- `ApiAdapterHandler`가 `/api/admin/*` 경로를 직접 `if/else`로 분기한다.
- 현재 구현은 대부분 `AdminHandler` 하나로 통합되어 있다.
- `ApiAdapterHandler`는 Admin API와 Mobile API를 상위에서 분기하고, Admin 세부 엔드포인트는 `AdminHandler`가 처리한다.
- `accounts`, `verifications`, `app-activations`, `members`, `system/users`도 현재 `AdminHandler` 범위에 포함되어 있다.

### 2.2 MobileHandler에서 따라야 할 패턴

`MobileHandler` 기준 공통 흐름은 다음과 같다.

1. `requestId` 생성
2. `normalizePath(path)` 호출
3. `CognitoAuthValidator.extractBearerToken(authHeader)`
4. `authValidator.validateToken(token)`
5. 컨텍스트 객체(`MobileRequestContext`) 생성
6. HTTP Method별 분기
7. 엔드포인트별 private 메서드 호출
8. 공통 성공/오류 응답 생성
9. `X-Request-Id` 헤더 포함

현재 구현도 이 흐름과 동일한 방향으로 정리되어 있다.

### 2.6 Members 도메인 정의

- `members`는 owner에 소속된 하위 운영 계정 도메인이다.
- `installer-admin-web` 로그인 대상이 아니라 `mobile app` 로그인 대상이다.
- 내부 운영 계정인 `system/users`와 구분된다.
- 실제 운전자 계정인 `drivers`와 구분된다.
- 저장은 기존 `app_user`를 재사용하되, 현재 구현/운영 축에서는 `user_type = 'STAFF'`를 member 도메인으로 해석한다.
- group 범위는 실제 운영 스키마의 `user_group_access`를 기준으로 한다.
- 저장 시 내부 값은 현재 `user_type = 'STAFF'`, `role = 'STAFF'` 축을 재사용하고, 외부 API 도메인명만 `members`로 노출한다.

### 2.7 System Users 도메인 정의

- `system/users`는 Installer Admin 내부 운영 계정 도메인이다.
- 실제 dev 운영 기준의 source of truth는 Aurora가 아니라 Cognito User Pool이다.
- `POST /api/admin/system/users`는 Cognito `AdminCreateUser` 기반으로 계정을 만들고 임시 비밀번호를 발급한다.
- 최초 로그인은 `NEW_PASSWORD_REQUIRED` 챌린지를 통해 새 비밀번호 설정을 강제한다.
- 역할은 Cognito group(`Admin`, `Operator`, `Installer`)을 기준으로 해석한다.

### 2.3 AuroraRepository 패턴

현재 `AuroraRepository`는 두 계열이 공존한다.

- 기존 admin/mobile 메서드: positional parameter 기반 `executeQuery`, `executeUpdate`, `executeCount`
- 신규 헬퍼: named parameter 기반 `executeQueryNew`, `executeUpdateNew`, `executeCountNew`

신규 Admin API 설계는 **`executeQueryNew` / `executeUpdateNew` / `executeCountNew` 우선**으로 정리한다.

### 2.4 CognitoAuthValidator 패턴

토큰 검증은 다음 환경변수를 사용한다.

- `COGNITO_USER_POOL_ID`
- `COGNITO_CLIENT_ID`
- `AWS_REGION`

검증 흐름은 다음과 같다.

1. Bearer 토큰 추출
2. JWKS fetch/caching
3. JWT signature 검증
4. audience 또는 `client_id` 검증
5. claims를 `AuthResult`로 반환

### 2.5 System.getenv() 사용 방식

현재 코드베이스는 환경변수를 직접 참조한다.

- `AuroraRepository`: `AURORA_CLUSTER_ARN`, `AURORA_SECRET_ARN`, `AURORA_DATABASE`, `AWS_REGION`
- `CognitoAuthValidator`: `COGNITO_USER_POOL_ID`, `COGNITO_CLIENT_ID`, `AWS_REGION`
- `CognitoUserManager`: `COGNITO_USER_POOL_ID`, `AWS_REGION`
- `MobileHandler`: `VEHICLE_STATE_TABLE_NAME`

AdminHandler는 직접 `System.getenv()`를 최소화하고, 기존 유틸/리포지토리 생성자에 위임하는 형태가 현재 패턴과 가장 잘 맞는다.

---

## 3. 권장 구조

### 3.1 목표 구조

- Lambda entrypoint는 계속 `ApiAdapterHandler`
- `/api/admin/*`, `/v1/admin/*` 경로는 `AdminHandler` 하나로 위임
- `AdminHandler` 내부에서 인증, 권한, 경로 분기, 공통 응답을 처리
- 데이터 접근은 `AdminRepository` 인터페이스 뒤로 숨기고, 실제 구현은 Aurora 기반으로 연결
- Cognito 계정 생성/초대가 필요한 API는 `CognitoUserManager`를 함께 사용

현재 코드 기준으로는 `AdminRepository` 인터페이스 추상화까지는 도입되지 않았고, `AdminHandler`가 `AuroraRepository`와 `CognitoUserManager`를 직접 조합해 사용한다.

### 3.2 ApiAdapterHandler 현재 구조

현재 구현은 아래 형태로 단순화되어 있다.

```java
if (normalizedPath.startsWith("/api/admin/") || normalizedPath.equals("/api/admin")) {
    String authHeader = event.getHeaders().get("Authorization");
    return adminHandler.handleRequest(path, method, queryParams, body, authHeader, objectMapper);
}
```

대시보드도 인증 대상이므로 `AdminHandler` 내부 인증 흐름으로 통합되어 있다.

---

## 4. AdminHandler.java 전체 구조와 메서드 시그니처

파일 경로:

`/Users/aircha/development/opencode/trackcar/aws-functions/src/main/java/com/trackcar/api/handlers/AdminHandler.java`

```java
package com.trackcar.api.handlers;

import com.amazonaws.services.lambda.runtime.events.APIGatewayProxyResponseEvent;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.trackcar.common.context.AdminRequestContext;
import com.trackcar.common.repository.AdminRepository;
import com.trackcar.common.repository.AuroraAdminRepository;
import com.trackcar.common.util.CognitoAuthValidator;
import com.trackcar.common.util.CognitoUserManager;

import java.util.Map;

public class AdminHandler {

    public AdminHandler();

    AdminHandler(
        AdminRepository adminRepository,
        CognitoAuthValidator authValidator,
        CognitoUserManager cognitoUserManager
    );

    public APIGatewayProxyResponseEvent handleRequest(
        String path,
        String method,
        Map<String, String> queryParams,
        Map<String, Object> body,
        String authHeader,
        ObjectMapper objectMapper
    );

    private APIGatewayProxyResponseEvent handleGetRequest(
        String path,
        Map<String, String> queryParams,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent handlePostRequest(
        String path,
        Map<String, Object> body,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent handlePutRequest(
        String path,
        Map<String, Object> body,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent getDashboard(
        Map<String, String> queryParams,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent listOwners(
        Map<String, String> queryParams,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent getOwner(
        String ownerId,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent createOwner(
        Map<String, Object> body,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent updateOwner(
        String ownerId,
        Map<String, Object> body,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent checkOwnerDuplicate(
        Map<String, Object> body,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent listGroups(
        Map<String, String> queryParams,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent createGroup(
        Map<String, Object> body,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent updateGroup(
        String groupId,
        Map<String, Object> body,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent listDevices(
        Map<String, String> queryParams,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent getDevice(
        String deviceId,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent createDevice(
        Map<String, Object> body,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent updateDevice(
        String deviceId,
        Map<String, Object> body,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent checkDeviceDuplicate(
        Map<String, Object> body,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent listVehicles(
        Map<String, String> queryParams,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent getVehicle(
        String vehicleId,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent createVehicle(
        Map<String, Object> body,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent updateVehicle(
        String vehicleId,
        Map<String, Object> body,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent listDrivers(
        Map<String, String> queryParams,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent listMembers(
        Map<String, String> queryParams,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent getMember(
        String memberId,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent createMember(
        Map<String, Object> body,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent updateMember(
        String memberId,
        Map<String, Object> body,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent inviteMember(
        String memberId,
        Map<String, Object> body,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent resendMemberInvite(
        String memberId,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent patchMemberStatus(
        String memberId,
        Map<String, Object> body,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent getDriver(
        String driverId,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent createDriver(
        Map<String, Object> body,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent updateDriver(
        String driverId,
        Map<String, Object> body,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent listAccountCandidates(
        Map<String, String> queryParams,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent linkDriverAccount(
        String driverId,
        Map<String, Object> body,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent createLinkedAccount(
        String driverId,
        Map<String, Object> body,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent getMappingCandidates(
        Map<String, String> queryParams,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent createMappingBundle(
        Map<String, Object> body,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent listVerifications(
        Map<String, String> queryParams,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent getVerification(
        String verificationId,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent runVerifications(
        Map<String, Object> body,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent listAppActivations(
        Map<String, String> queryParams,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent getAppActivation(
        String activationId,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent inviteAppActivation(
        String activationId,
        Map<String, Object> body,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent resendAppActivationInvite(
        String activationId,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent markAppActivationLive(
        String activationId,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent listSystemUsers(
        Map<String, String> queryParams,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private APIGatewayProxyResponseEvent createSystemUser(
        Map<String, Object> body,
        AdminRequestContext context,
        ObjectMapper objectMapper,
        String requestId
    );

    private AdminRequestContext buildContext(CognitoAuthValidator.AuthResult authResult, String requestId);
    private String normalizePath(String path);
    private String extractPathParam(String path, String prefix, String suffix);
    private int parseIntOrDefault(String value, int defaultValue);
    private java.util.List<String> parseCsvOrRepeatedParam(Map<String, String> queryParams, String key);
    private boolean hasAnyRole(AdminRequestContext context, String... roles);
    private APIGatewayProxyResponseEvent unauthorizedResponse(String code, String message, ObjectMapper objectMapper, String requestId);
    private APIGatewayProxyResponseEvent forbiddenResponse(String message, ObjectMapper objectMapper, String requestId);
    private APIGatewayProxyResponseEvent badRequestResponse(String code, String message, ObjectMapper objectMapper, String requestId);
    private APIGatewayProxyResponseEvent notFound(ObjectMapper objectMapper, String requestId);
    private java.util.Map<String, Object> createSuccessResponse(Object data, String requestId);
    private java.util.Map<String, Object> createErrorResponse(String code, String message, Object details, String requestId);
    private APIGatewayProxyResponseEvent buildResponse(int statusCode, java.util.Map<String, Object> body, ObjectMapper objectMapper, String requestId);
}
```

### 4.1 보조 컨텍스트 클래스

파일 경로:

`/Users/aircha/development/opencode/trackcar/aws-functions/src/main/java/com/trackcar/common/context/AdminRequestContext.java`

```java
@Data
@Builder
public class AdminRequestContext {
    private final String userId;
    private final String username;
    private final String name;
    private final String email;
    private final String role;
    private final java.util.List<String> groups;
    private final String requestId;

    public boolean isInstaller();
    public boolean isOperator();
    public boolean isAdmin();
    public boolean hasAnyRole(String... roles);
}
```

`MobileRequestContext`와 동일한 Lombok 스타일을 유지하되, Admin 역할 체크에 맞춘 helper만 추가한다.

---

## 5. 엔드포인트별 매핑

역할 규칙은 아래 기준으로 정리한다.

- 스펙에 명시된 경우: 그 값을 그대로 사용
- 스펙에 명시가 없는 경우: 문서 1.2의 공통 규칙인 `Installer, Operator, Admin` 허용으로 처리

### 5.1 Dashboard API

| Method | Path | Handler method | Request DTO | Response DTO | Role |
|---|---|---|---|---|---|
| GET | `/api/admin/dashboard` | `getDashboard(...)` | `DashboardQuery` | `DashboardResponseData` | Installer, Operator, Admin |

### 5.2 Owner API

| Method | Path | Handler method | Request DTO | Response DTO | Role |
|---|---|---|---|---|---|
| GET | `/api/admin/owners` | `listOwners(...)` | `OwnerListQuery` | `OwnerListResponseData` | Installer, Operator, Admin |
| GET | `/api/admin/owners/{ownerId}` | `getOwner(...)` | `OwnerDetailPath` | `OwnerDetailResponseData` | Installer, Operator, Admin |
| POST | `/api/admin/owners` | `createOwner(...)` | `CreateOwnerRequest` | `CreateOwnerResponseData` | Installer, Operator, Admin |
| PUT | `/api/admin/owners/{ownerId}` | `updateOwner(...)` | `UpdateOwnerRequest` | `UpdateOwnerResponseData` | Operator, Admin |
| POST | `/api/admin/owners/check-duplicate` | `checkOwnerDuplicate(...)` | `OwnerDuplicateCheckRequest` | `DuplicateCheckResponseData` | Installer, Operator, Admin |

### 5.3 Group API

| Method | Path | Handler method | Request DTO | Response DTO | Role |
|---|---|---|---|---|---|
| GET | `/api/admin/groups` | `listGroups(...)` | `GroupListQuery` | `GroupListResponseData` | Installer, Operator, Admin |
| POST | `/api/admin/groups` | `createGroup(...)` | `CreateGroupRequest` | `CreateGroupResponseData` | Installer, Operator, Admin |
| PUT | `/api/admin/groups/{groupId}` | `updateGroup(...)` | `UpdateGroupRequest` | `UpdateGroupResponseData` | Operator, Admin |

### 5.4 Device API

| Method | Path | Handler method | Request DTO | Response DTO | Role |
|---|---|---|---|---|---|
| GET | `/api/admin/devices` | `listDevices(...)` | `DeviceListQuery` | `DeviceListResponseData` | Installer, Operator, Admin |
| GET | `/api/admin/devices/{deviceId}` | `getDevice(...)` | `DeviceDetailPath` | `DeviceDetailResponseData` | Installer, Operator, Admin |
| POST | `/api/admin/devices` | `createDevice(...)` | `CreateDeviceRequest` | `CreateDeviceResponseData` | Installer, Operator, Admin |
| PUT | `/api/admin/devices/{deviceId}` | `updateDevice(...)` | `UpdateDeviceRequest` | `UpdateDeviceResponseData` | Operator, Admin |
| POST | `/api/admin/devices/check-duplicate` | `checkDeviceDuplicate(...)` | `DeviceDuplicateCheckRequest` | `DuplicateCheckResponseData` | Installer, Operator, Admin |

### 5.5 Vehicle API

| Method | Path | Handler method | Request DTO | Response DTO | Role |
|---|---|---|---|---|---|
| GET | `/api/admin/vehicles` | `listVehicles(...)` | `VehicleListQuery` | `VehicleListResponseData` | Installer, Operator, Admin |
| GET | `/api/admin/vehicles/{vehicleId}` | `getVehicle(...)` | `VehicleDetailPath` | `VehicleDetailResponseData` | Installer, Operator, Admin |
| POST | `/api/admin/vehicles` | `createVehicle(...)` | `CreateVehicleRequest` | `CreateVehicleResponseData` | Installer, Operator, Admin |
| PUT | `/api/admin/vehicles/{vehicleId}` | `updateVehicle(...)` | `UpdateVehicleRequest` | `UpdateVehicleResponseData` | Installer, Operator, Admin |

### 5.6 Driver API

| Method | Path | Handler method | Request DTO | Response DTO | Role |
|---|---|---|---|---|---|
| GET | `/api/admin/drivers` | `listDrivers(...)` | `DriverListQuery` | `DriverListResponseData` | Installer, Operator, Admin |
| GET | `/api/admin/drivers/{driverId}` | `getDriver(...)` | `DriverDetailPath` | `DriverDetailResponseData` | Installer, Operator, Admin |
| POST | `/api/admin/drivers` | `createDriver(...)` | `CreateDriverRequest` | `CreateDriverResponseData` | Installer, Operator, Admin |
| PUT | `/api/admin/drivers/{driverId}` | `updateDriver(...)` | `UpdateDriverRequest` | `UpdateDriverResponseData` | Installer, Operator, Admin |

### 5.7 Account Linking API

| Method | Path | Handler method | Request DTO | Response DTO | Role |
|---|---|---|---|---|---|
| GET | `/api/admin/accounts/candidates` | `listAccountCandidates(...)` | `AccountCandidateListQuery` | `AccountCandidateListResponseData` | Installer, Operator, Admin |
| POST | `/api/admin/drivers/{driverId}/link-account` | `linkDriverAccount(...)` | `LinkDriverAccountRequest` | `LinkDriverAccountResponseData` | Installer, Operator, Admin |
| POST | `/api/admin/drivers/{driverId}/create-linked-account` | `createLinkedAccount(...)` | `CreateLinkedAccountRequest` | `CreateLinkedAccountResponseData` | Installer, Operator, Admin |

### 5.8 Mapping API

| Method | Path | Handler method | Request DTO | Response DTO | Role |
|---|---|---|---|---|---|
| GET | `/api/admin/mappings/candidates` | `getMappingCandidates(...)` | `MappingCandidateQuery` | `MappingCandidateResponseData` | Installer, Operator, Admin |
| POST | `/api/admin/mappings/bundle` | `createMappingBundle(...)` | `CreateMappingBundleRequest` | `CreateMappingBundleResponseData` | Installer, Operator, Admin |

### 5.9 Verification API

| Method | Path | Handler method | Request DTO | Response DTO | Role |
|---|---|---|---|---|---|
| GET | `/api/admin/verifications` | `listVerifications(...)` | `VerificationListQuery` | `VerificationListResponseData` | Installer, Operator, Admin |
| GET | `/api/admin/verifications/{verificationId}` | `getVerification(...)` | `VerificationDetailPath` | `VerificationDetailResponseData` | Installer, Operator, Admin |
| POST | `/api/admin/verifications/run` | `runVerifications(...)` | `RunVerificationsRequest` | `RunVerificationsResponseData` | Installer, Operator, Admin |

### 5.10 App Activation API

| Method | Path | Handler method | Request DTO | Response DTO | Role |
|---|---|---|---|---|---|
| GET | `/api/admin/app-activations` | `listAppActivations(...)` | `AppActivationListQuery` | `AppActivationListResponseData` | Installer, Operator, Admin |
| GET | `/api/admin/app-activations/{activationId}` | `getAppActivation(...)` | `AppActivationDetailPath` | `AppActivationDetailResponseData` | Installer, Operator, Admin |
| POST | `/api/admin/app-activations/{activationId}/invite` | `inviteAppActivation(...)` | `InviteAppActivationRequest` | `InviteAppActivationResponseData` | Installer, Operator, Admin |
| POST | `/api/admin/app-activations/{activationId}/resend-invite` | `resendAppActivationInvite(...)` | `ResendAppActivationInvitePath` | `InviteAppActivationResponseData` | Installer, Operator, Admin |
| POST | `/api/admin/app-activations/{activationId}/mark-live` | `markAppActivationLive(...)` | `MarkAppActivationLivePath` | `MarkAppActivationLiveResponseData` | Installer, Operator, Admin |

### 5.11 System API

| Method | Path | Handler method | Request DTO | Response DTO | Role |
|---|---|---|---|---|---|
| GET | `/api/admin/system/users` | `listSystemUsers(...)` | `SystemUserListQuery` | `SystemUserListResponseData` | Admin |
| POST | `/api/admin/system/users` | `createSystemUser(...)` | `CreateSystemUserRequest` | `CreateSystemUserResponseData` | Admin |

---

## 6. DTO 클래스 정의

패키지 제안:

`/Users/aircha/development/opencode/trackcar/aws-functions/src/main/java/com/trackcar/api/dto/admin`

스타일 제안:

- `@Data`
- `@Builder`
- `@NoArgsConstructor`
- `@AllArgsConstructor`

현재 프로젝트가 Lombok 기반 보조 모델을 이미 사용하므로 같은 스타일이 자연스럽다.

### 6.1 공통 DTO

```java
public class PageInfoDto {
    private Integer page;
    private Integer size;
    private Long total;
    private Boolean hasNext;
}

public class DuplicateCheckResponseData {
    private Boolean duplicated;
    private java.util.List<String> duplicatedFields;
}
```

### 6.2 Dashboard DTO

```java
public class DashboardQuery {
    private String dateFrom;
    private String dateTo;
    private String ownerType;
    private String groupId;
    private String deviceModel;
    private java.util.List<String> verificationStatus;
}

public class DashboardResponseData {
    private DashboardMetricsDto metrics;
    private DashboardChartsDto charts;
    private DashboardListsDto lists;
}

public class DashboardMetricsDto {
    private Long installedVehicleCount;
    private Long onlineDeviceCount;
    private Long unlinkedDeviceCount;
    private Long verificationPassCount;
}

public class DashboardChartsDto {
    private java.util.List<InstallTrendPointDto> installTrend;
    private DeviceStatusChartDto deviceStatus;
    private VehicleDriverMappingChartDto vehicleDriverMapping;
    private java.util.List<ActivationFunnelStepDto> activationFunnel;
}

public class InstallTrendPointDto {
    private String date;
    private Long count;
}

public class DeviceStatusChartDto {
    private Long ONLINE;
    private Long OFFLINE;
    private Long NEVER_CONNECTED;
    private Long ERROR;
}

public class VehicleDriverMappingChartDto {
    private Long MAPPED;
    private Long VEHICLE_ONLY;
    private Long DRIVER_ONLY;
    private Long UNMAPPED;
}

public class ActivationFunnelStepDto {
    private String step;
    private Long count;
    private Integer percent;
}

public class DashboardListsDto {
    private java.util.List<RecentIssueVehicleDto> recentIssueVehicles;
}

public class RecentIssueVehicleDto {
    private String vehicleId;
    private String vehicleNo;
    private String deviceSerialNo;
    private String vehicleStatus;
    private String installationStatus;
    private String severity;
    private String lastReceivedAt;
    private String locationSummary;
}
```

### 6.3 Owner DTO

```java
public class OwnerListQuery {
    private String keyword;
    private String ownerType;
    private String status;
    private String groupId;
    private Integer page;
    private Integer size;
    private String sort;
}

public class OwnerListResponseData {
    private java.util.List<OwnerSummaryDto> items;
    private PageInfoDto pageInfo;
}

public class OwnerSummaryDto {
    private String ownerId;
    private String ownerType;
    private String nameOrCompanyName;
    private String phone;
    private String businessNo;
    private String transportBizNo;
    private String defaultGroupName;
    private Integer vehicleCount;
    private Integer driverCount;
    private Integer unmappedVehicleCount; // device_binding 또는 driver_link가 없는 차량 수
    private String appActivationStatus;
    private String createdAt;
}

public class OwnerDetailResponseData {
    private OwnerDetailDto owner;
}

public class OwnerDetailDto {
    private String ownerId;
    private String ownerType;
    private String nameOrCompanyName;
    private String phone;
    private String email;
    private String businessNo;
    private String transportBizNo;
    private String defaultGroupName;
    private String status;
    private String createdAt;
    private String updatedAt;
}

public class CreateOwnerRequest {
    private String ownerType;
    private String nameOrCompanyName;
    private String phone;
    private String email;
    private String businessNo;
    private String transportBizNo;
    private Boolean createDefaultGroup;
    private String defaultGroupName;
    private String inviteMethod;
}

public class CreateOwnerResponseData {
    private String ownerId;
    private String defaultGroupId;
}

public class UpdateOwnerRequest {
    private String nameOrCompanyName;
    private String phone;
    private String email;
    private String businessNo;
    private String transportBizNo;
    private String status;
}

public class UpdateOwnerResponseData {
    private String ownerId;
}

public class OwnerDuplicateCheckRequest {
    private String ownerType;
    private String phone;
    private String businessNo;
}
```

### 6.4 Group DTO

```java
public class GroupListQuery {
    private String ownerId;
    private String keyword;
    private String status;
    private Integer page;
    private Integer size;
    private String sort;
}

public class GroupListResponseData {
    private java.util.List<GroupSummaryDto> items;
    private PageInfoDto pageInfo;
}

public class GroupSummaryDto {
    private String groupId;
    private String ownerName;
    private String groupName;
    private Integer memberCount;
    private Integer vehicleCount;
    private Integer driverCount;
    private String status;
    private Boolean isDefaultGroup;
    private String createdAt;
}

public class CreateGroupRequest {
    private String ownerId;
    private String groupName;
    private String status;
    private Boolean isDefaultGroup;
}

public class CreateGroupResponseData {
    private String groupId;
}

public class UpdateGroupRequest {
    private String groupName;
    private String status;
    private Boolean isDefaultGroup;
}

public class UpdateGroupResponseData {
    private String groupId;
}
```

`UpdateGroupResponseData`는 스펙에 본문 예시가 없으므로, 현재 `GroupHandler`의 id 반환 패턴을 따른다.

### 6.5 Device DTO

```java
public class DeviceListQuery {
    private String keyword;
    private String ownerId;
    private java.util.List<String> status;
    private String installationStatus;
    private Integer page;
    private Integer size;
    private String sort;
}

public class DeviceListResponseData {
    private java.util.List<DeviceSummaryDto> items;
    private PageInfoDto pageInfo;
}

public class DeviceSummaryDto {
    private String deviceId;
    private String serialNo;
    private String approvalNo;
    private String productSerialNo;
    private String modelName;
    private String deviceStatus;
    private String installationStatus;
    private String vehicleNo;
    private String lastReceivedAt;
    private String locationSummary;
}

public class DeviceDetailResponseData {
    private DeviceDetailDto device;
    private java.util.List<DeviceStatusHistoryDto> statusHistory;
}

public class DeviceDetailDto {
    private String deviceId;
    private String serialNo;
    private String approvalNo;
    private String productSerialNo;
    private String modelName;
    private String firmwareVersion;
    private String lineNumber;
    private String deviceStatus;
    private String installationStatus;
    private String lastReceivedAt;
    private String locationSummary;
    private String installedAt;
    private String installNote;
}

public class DeviceStatusHistoryDto {
    private String status;
    private String changedAt;
    private String changedBy;
}

public class CreateDeviceRequest {
    private String serialNo;
    private String approvalNo;
    private String productSerialNo;
    private String modelName;
    private String firmwareVersion;
    private String lineNumber;
    private String installationStatus;
    private String installNote;
}

public class CreateDeviceResponseData {
    private String deviceId;
}

public class UpdateDeviceRequest {
    private String modelName;
    private String firmwareVersion;
    private String lineNumber;
    private String installationStatus;
    private String installNote;
}

public class UpdateDeviceResponseData {
    private String deviceId;
}

public class DeviceDuplicateCheckRequest {
    private String serialNo;
    private String productSerialNo;
}
```

`UpdateDeviceResponseData`와 `DuplicateCheckResponseData`는 스펙 본문이 생략되어 있어 현재 `DeviceHandler`/`AuroraRepository.checkDeviceDuplicate(...)` 패턴에 맞춘다.

### 6.6 Vehicle DTO

```java
public class VehicleListQuery {
    private String keyword;
    private String ownerId;
    private String groupId;
    private java.util.List<String> vehicleStatus;
    private java.util.List<String> runningStatus;
    private Integer page;
    private Integer size;
    private String sort;
}

public class VehicleListResponseData {
    private java.util.List<VehicleSummaryDto> items;
    private PageInfoDto pageInfo;
}

public class VehicleSummaryDto {
    private String vehicleId;
    private String vehicleNo;
    private String vehicleType;
    private String vin;
    private String ownerName;
    private String groupName;
    private String vehicleStatus;
    private String runningStatus;
    private String installationStatus;
    private String deviceSerialNo;
    private String primaryDriverName;
    private String lastReceivedAt;
    private String locationSummary;
}

public class VehicleDetailResponseData {
    private VehicleDetailDto vehicle;
}

public class VehicleDetailDto {
    private String vehicleId;
    private String vehicleNo;
    private String vehicleType;
    private String vin;
    private String ownerId;
    private String ownerName;
    private String groupId;
    private String groupName;
    private String vehicleStatus;
    private String runningStatus;
    private String installationStatus;
    private String lastReceivedAt;
    private String locationSummary;
    private String createdAt;
}

public class CreateVehicleRequest {
    private String vehicleNo;
    private String vin;
    private String vehicleType;
    private String ownerId;
    private String groupId;
}

public class CreateVehicleResponseData {
    private String vehicleId;
}

public class UpdateVehicleRequest {
    private String vehicleNo;
    private String vin;
    private String vehicleType;
    private String groupId;
}

public class UpdateVehicleResponseData {
    private String vehicleId;
}
```

`UpdateVehicleResponseData`는 스펙 예시가 없으므로 현재 `VehicleHandler`의 id 반환 패턴을 따른다.

### 6.7 Driver DTO

```java
public class DriverListQuery {
    private String keyword;
    private String ownerId;
    private String groupId;
    private String accountLinked;
    private String status;
    private Integer page;
    private Integer size;
    private String sort;
}

public class DriverListResponseData {
    private java.util.List<DriverSummaryDto> items;
    private PageInfoDto pageInfo;
}

public class DriverSummaryDto {
    private String driverId;
    private String driverName;
    private String phone;
    private String driverCode;
    private String ownerName;
    private String groupName;
    private String primaryVehicleNo;
    private String accountLinked;
    private String appActivationStatus;
    private String status;
}

public class DriverDetailResponseData {
    private DriverDetailDto driver;
}

public class DriverDetailDto {
    private String driverId;
    private String driverName;
    private String phone;
    private String driverCode;
    private String ownerId;
    private String ownerName;
    private String groupId;
    private String groupName;
    private String primaryVehicleNo;
    private String accountLinked;
    private String appActivationStatus;
    private String status;
    private String createdAt;
}

public class CreateDriverRequest {
    private String driverName;
    private String phone;
    private String driverCode;
    private String ownerId;
    private String groupId;
}

public class CreateDriverResponseData {
    private String driverId;
}

public class UpdateDriverRequest {
    private String driverName;
    private String phone;
    private String driverCode;
    private String groupId;
    private String status;
}

public class UpdateDriverResponseData {
    private String driverId;
}
```

`CreateDriverResponseData`와 `UpdateDriverResponseData`는 스펙 응답 예시가 없으므로 현재 `DriverHandler`의 id 반환 패턴을 따른다.

### 6.8 Account Linking DTO

```java
public class AccountCandidateListQuery {
    private String keyword;
    private String ownerId;
    private String groupId;
}

public class AccountCandidateListResponseData {
    private java.util.List<AccountCandidateDto> items;
    private PageInfoDto pageInfo;
}

public class AccountCandidateDto {
    private String userId;
    private String userType;
    private String userName;
    private String phone;
    private String ownerName;
    private String appActivationStatus;
}

public class LinkDriverAccountRequest {
    private String userId;
}

public class LinkDriverAccountResponseData {
    private String driverId;
    private String userId;
    private Boolean linked;
}

public class CreateLinkedAccountRequest {
    private String userType;
    private String userName;
    private String phone;
    private Boolean inviteNow;
}

public class CreateLinkedAccountResponseData {
    private String driverId;
    private String userId;
    private String inviteStatus;
}
```

`accounts/candidates`는 응답에 `page_info`가 있으나 스펙 요청 파라미터에는 `page`, `size`가 없다. DTO는 스펙 우선으로 query에 pagination 필드를 두지 않고, 응답의 `pageInfo`는 optional로 유지한다.

### 6.9 Mapping DTO

```java
public class MappingCandidateQuery {
    private String ownerId;
    private String groupId;
    private String keyword;
}

public class MappingCandidateResponseData {
    private java.util.List<MappingVehicleCandidateDto> vehicles;
    private java.util.List<MappingDriverCandidateDto> drivers;
    private java.util.List<MappingDeviceCandidateDto> devices;
}

public class MappingVehicleCandidateDto {
    private String vehicleId;
    private String vehicleNo;
    private String vehicleStatus;
    private Boolean isMapped;
}

public class MappingDriverCandidateDto {
    private String driverId;
    private String driverName;
    private String status;
    private Boolean isMapped;
}

public class MappingDeviceCandidateDto {
    private String deviceId;
    private String serialNo;
    private String deviceStatus;
    private Boolean isMapped;
}

public class CreateMappingBundleRequest {
    private String ownerId;
    private String groupId;
    private String vehicleId;
    private String driverId;
    private String deviceId;
    private Boolean driverAccountChecked;
}

public class CreateMappingBundleResponseData {
    private String deviceVehicleMappingId;
    private String vehicleDriverMappingId;
}
```

### 6.10 Verification DTO

```java
public class VerificationListQuery {
    private String keyword;
    private java.util.List<String> resultStatus;
    private java.util.List<String> severity;
    private String ownerId;
    private Integer page;
    private Integer size;
    private String sort;
}

public class VerificationListResponseData {
    private java.util.List<VerificationSummaryDto> items;
    private PageInfoDto pageInfo;
}

public class VerificationSummaryDto {
    private String verificationId;
    private String targetType;
    private String vehicleNo;
    private String deviceSerialNo;
    private String driverName;
    private Boolean gpsReceived;
    private Boolean heartbeatReceived;
    private Boolean appLoginChecked;
    private String resultStatus;
    private String severity;
    private String checkedAt;
}

public class VerificationDetailResponseData {
    private VerificationDetailDto verification;
}

public class VerificationDetailDto {
    private String verificationId;
    private String targetType;
    private String targetId;
    private String vehicleNo;
    private String resultStatus;
    private String severity;
    private VerificationCheckItemsDto checkItems;
    private String lastReceivedAt;
    private String locationSummary;
    private String checkedAt;
}

public class VerificationCheckItemsDto {
    private String deviceRegistered;
    private String vehicleMapped;
    private String driverMapped;
    private String gpsReceived;
    private String heartbeatReceived;
    private String appInvited;
    private String firstLoginDone;
}

public class RunVerificationsRequest {
    private String targetType;
    private java.util.List<String> targetIds;
}

public class RunVerificationsResponseData {
    private String requestId;
    private Integer acceptedCount;
}
```

### 6.11 App Activation DTO

```java
public class AppActivationListQuery {
    private String keyword;
    private java.util.List<String> appActivationStatus;
    private java.util.List<String> userType;
    private String ownerId;
    private Integer page;
    private Integer size;
    private String sort;
}

public class AppActivationListResponseData {
    private java.util.List<AppActivationSummaryDto> items;
    private PageInfoDto pageInfo;
}

public class AppActivationSummaryDto {
    private String activationId;
    private String userName;
    private String userType;
    private String phone;
    private String ownerName;
    private String vehicleNo;
    private String appActivationStatus;
    private String invitedAt;
    private String firstLoginAt;
    private String liveAt;
}

public class AppActivationDetailResponseData {
    private AppActivationDetailDto activation;
    private java.util.List<AppInviteHistoryDto> inviteHistory;
}

public class AppActivationDetailDto {
    private String activationId;
    private String userId;
    private String userName;
    private String userType;
    private String phone;
    private String ownerName;
    private String appActivationStatus;
    private String linkedVehicleNo;
    private String linkedDeviceSerialNo;
}

public class AppInviteHistoryDto {
    private String inviteMethod;
    private String invitedAt;
    private String invitedBy;
}

public class InviteAppActivationRequest {
    private String method;
}

public class InviteAppActivationResponseData {
    private String inviteStatus;
    private String invitedAt;
}

public class MarkAppActivationLiveResponseData {
    private String appActivationStatus;
}
```

### 6.12 System User DTO

```java
public class SystemUserListQuery {
    private String keyword;
    private java.util.List<String> role;
    private String status;
    private Integer page;
    private Integer size;
    private String sort;
}

public class SystemUserListResponseData {
    private java.util.List<SystemUserSummaryDto> items;
    private PageInfoDto pageInfo;
}

public class SystemUserSummaryDto {
    private String userId;
    private String userName;
    private String email;
    private String role;
    private String status;
    private String lastLoginAt;
    private String createdAt;
}

public class CreateSystemUserRequest {
    private String userName;
    private String email;
    private String role;
    private String status;
}

public class CreateSystemUserResponseData {
    private String userId;
    private String loginUsername;
    private String temporaryPassword;
    private Boolean passwordChangeRequired;
    private String role;
    private String status;
}
```

### 6.13 Members DTO

```java
public class MemberListQuery {
    private String keyword;
    private String ownerId;
    private String groupId;
    private String role;
    private String appActivationStatus;
    private String status;
    private Integer page;
    private Integer size;
    private String sort;
}

public class MemberListResponseData {
    private java.util.List<MemberSummaryDto> items;
    private PageInfoDto pageInfo;
}

public class MemberSummaryDto {
    private String memberId;
    private String memberName;
    private String phone;
    private String email;
    private String ownerId;
    private String ownerName;
    private String groupId;
    private String groupName;
    private String role;
    private String appActivationStatus;
    private String status;
    private String createdAt;
}

public class MemberDetailResponseData {
    private MemberDetailDto member;
}

public class MemberDetailDto extends MemberSummaryDto {
}

public class CreateMemberRequest {
    private String memberName;
    private String phone;
    private String email;
    private String ownerId;
    private String groupId;
    private String role;
    private Boolean inviteNow;
}

public class CreateMemberResponseData {
    private String memberId;
    private String userId;
    private String appActivationStatus;
}

public class UpdateMemberRequest {
    private String memberName;
    private String phone;
    private String email;
    private String groupId;
    private String role;
    private String status;
}

public class InviteMemberRequest {
    private String method;
}

public class InviteMemberResponseData {
    private String memberId;
    private String appActivationStatus;
    private String invitedAt;
}

public class PatchMemberStatusRequest {
    private String status;
}
```

---

## 7. AdminRepository 인터페이스와 Aurora 연동 방식

### 7.1 인터페이스 경로

`/Users/aircha/development/opencode/trackcar/aws-functions/src/main/java/com/trackcar/common/repository/AdminRepository.java`

```java
package com.trackcar.common.repository;

import java.util.List;
import java.util.Map;

public interface AdminRepository {

    AuroraRepository.AdminQueryResult findOwners(String keyword, String ownerType, String status, String groupId, int page, int size, String sort);
    Map<String, Object> findOwnerById(String ownerId);
    String createOwner(String name, String ownerType, String phone, String email, String businessNo, String transportBizNo);
    void updateOwner(String ownerId, String name, String phone, String email, String businessNo, String transportBizNo, String status);
    Map<String, Object> checkOwnerDuplicate(String phone, String businessNo);

    AuroraRepository.AdminQueryResult findGroups(String ownerId, String keyword, String status, int page, int size, String sort);
    String createGroup(String ownerId, String groupName, String description, String status, Boolean isDefaultGroup);
    void updateGroup(String groupId, String groupName, String description, String status, Boolean isDefaultGroup);

    AuroraRepository.AdminQueryResult findMembers(String keyword, String ownerId, String groupId, String role, String appActivationStatus, String status, int page, int size, String sort);
    Map<String, Object> findMemberById(String memberId);
    Map<String, Object> createMember(String memberName, String phone, String email, String ownerId, String groupId, String role, Boolean inviteNow, String createdBy);
    void updateMember(String memberId, String memberName, String phone, String email, String groupId, String role, String status, String updatedBy);
    Map<String, Object> inviteMember(String memberId, String method, String invitedBy);
    Map<String, Object> resendMemberInvite(String memberId, String invitedBy);
    void updateMemberStatus(String memberId, String status, String updatedBy);

    AuroraRepository.AdminQueryResult findDevices(String keyword, String ownerId, List<String> statuses, String installStatus, int page, int size, String sort);
    Map<String, Object> findDeviceById(String deviceId);
    String createDevice(String serialNo, String approvalNo, String productSerialNo, String modelName, String firmwareVersion, String lineNumber, String installStatus, String installNote);
    void updateDevice(String deviceId, String modelName, String firmwareVersion, String lineNumber, String installStatus, String installNote);
    Map<String, Object> checkDeviceDuplicate(String serialNo, String productSerialNo);

    AuroraRepository.AdminQueryResult findVehicles(String keyword, String ownerId, String groupId, List<String> vehicleStatuses, List<String> runningStatuses, int page, int size, String sort);
    Map<String, Object> findVehicleById(String vehicleId);
    String createVehicle(String vehicleNo, String vin, String vehicleType, String ownerId, String groupId);
    void updateVehicle(String vehicleId, String vehicleNo, String vin, String vehicleType, String groupId);

    AuroraRepository.AdminQueryResult findDrivers(String keyword, String ownerId, String groupId, String accountLinked, String status, int page, int size, String sort);
    Map<String, Object> findDriverById(String driverId);
    String createDriver(String name, String phone, String driverCode, String ownerId, String groupId);
    void updateDriver(String driverId, String name, String phone, String driverCode, String groupId, String status);

    AuroraRepository.AdminQueryResult findAccountCandidates(String keyword, String ownerId, String groupId, int page, int size);
    void linkDriverAccount(String driverId, String userId, String linkedBy);
    String createLinkedAccount(String driverId, String userType, String userName, String phone, String cognitoUserId, Boolean inviteNow, String createdBy);

    Map<String, Object> findMappingCandidates(String ownerId, String groupId, String keyword);
    Map<String, String> createBundleMapping(String ownerId, String groupId, String vehicleId, String driverId, String deviceId, Boolean driverAccountChecked, String mappedBy);

    Map<String, Object> getDashboard(String dateFrom, String dateTo, String ownerType, String groupId, String deviceModel, List<String> verificationStatuses);

    AuroraRepository.AdminQueryResult findVerifications(String keyword, List<String> resultStatuses, List<String> severities, String ownerId, int page, int size, String sort);
    Map<String, Object> findVerificationById(String verificationId);
    Map<String, Object> requestVerifications(String targetType, List<String> targetIds, String requestedBy);

    AuroraRepository.AdminQueryResult findAppActivations(String keyword, List<String> activationStatuses, List<String> userTypes, String ownerId, int page, int size, String sort);
    Map<String, Object> findAppActivationById(String activationId);
    Map<String, Object> inviteAppActivation(String activationId, String method, String invitedBy);
    Map<String, Object> resendAppActivationInvite(String activationId, String invitedBy);
    Map<String, Object> markAppActivationLive(String activationId, String updatedBy);

    // System users are served from Cognito in current implementation.
    // AuroraRepository is not the source of truth for system users.
}
```

### 7.2 구현체 방향

구현체 경로:

`/Users/aircha/development/opencode/trackcar/aws-functions/src/main/java/com/trackcar/common/repository/AuroraAdminRepository.java`

```java
public class AuroraAdminRepository implements AdminRepository {

    private final AuroraRepository auroraRepository;

    public AuroraAdminRepository() {
        this.auroraRepository = new AuroraRepository();
    }

    AuroraAdminRepository(AuroraRepository auroraRepository) {
        this.auroraRepository = auroraRepository;
    }

    // 기존 구현이 있는 owner/group/device/vehicle/driver/mapping/dashboard는 위임
// 신규 account/verification/app-activation은 executeQueryNew/executeUpdateNew/executeCountNew 사용
// system-user는 CognitoUserManager 기반으로 처리
}
```

### 7.3 Aurora helper 사용 규칙

신규 SQL은 다음 패턴을 따른다.

```java
String sql = """
    SELECT user_id, name, phone
    FROM app_user
    WHERE (:ownerId IS NULL OR organization_id = :ownerId)
      AND deleted_at IS NULL
""";

List<Map<String, Object>> rows = auroraRepository.executeQueryNew(
    sql,
    "ownerId", ownerId
);
```

```java
auroraRepository.executeUpdateNew(
    "UPDATE app_user SET status = :status WHERE user_id = :userId",
    "status", status,
    "userId", userId
);
```

```java
long total = auroraRepository.executeCountNew(
    "SELECT COUNT(*) FROM app_user WHERE deleted_at IS NULL AND role = :role",
    "role", "DRIVER"
);
```

### 7.4 Cognito 연동이 필요한 엔드포인트

아래 엔드포인트는 `AdminRepository`만으로 끝나지 않고 `CognitoUserManager`를 함께 사용한다.

- `POST /api/admin/drivers/{driverId}/create-linked-account`
- `POST /api/admin/app-activations/{activationId}/invite`
- `POST /api/admin/app-activations/{activationId}/resend-invite`
- `GET /api/admin/system/users`
- `POST /api/admin/system/users`

권장 순서는 다음과 같다.

1. 요청 검증
2. Cognito 작업 수행 (`inviteUser`, `listUsers`, `adminListGroupsForUser`)
3. 필요 시 상태 반영 (`disableUser`, `enableUser`)
4. API 응답 DTO 반환

---

## 8. 단위 테스트 구조

현재 `build.gradle`에는 `mockito-core`는 있으나 JUnit Jupiter 의존성은 없다. 실제 테스트 구현 시에는 JUnit 5 dependency 추가가 필요하다.

### 8.1 테스트 파일 구조

```text
/Users/aircha/development/opencode/trackcar/aws-functions/src/test/java/com/trackcar/api/handlers/AdminHandlerTest.java
/Users/aircha/development/opencode/trackcar/aws-functions/src/test/java/com/trackcar/common/context/AdminRequestContextTest.java
/Users/aircha/development/opencode/trackcar/aws-functions/src/test/java/com/trackcar/common/repository/AuroraAdminRepositoryTest.java
```

### 8.2 AdminHandlerTest 구조

```java
class AdminHandlerTest {

    private AdminRepository adminRepository;
    private CognitoAuthValidator authValidator;
    private CognitoUserManager cognitoUserManager;
    private ObjectMapper objectMapper;
    private AdminHandler adminHandler;

    @Nested class AuthenticationTests { }
    @Nested class AuthorizationTests { }
    @Nested class DashboardRouteTests { }
    @Nested class OwnerRouteTests { }
    @Nested class GroupRouteTests { }
    @Nested class DeviceRouteTests { }
    @Nested class VehicleRouteTests { }
    @Nested class DriverRouteTests { }
    @Nested class AccountLinkingRouteTests { }
    @Nested class MappingRouteTests { }
    @Nested class VerificationRouteTests { }
    @Nested class AppActivationRouteTests { }
    @Nested class SystemUserRouteTests { }
    @Nested class ErrorResponseTests { }
}
```

### 8.3 최소 검증 시나리오

#### AuthenticationTests

- Authorization 헤더 없음 -> 401
- Bearer 토큰 파싱 실패 -> 401
- `authValidator.validateToken(...)` 실패 -> 401

#### AuthorizationTests

- `Operator, Admin` 전용 엔드포인트에 `Installer` 접근 -> 403
- `Admin` 전용 엔드포인트에 `Operator` 접근 -> 403

#### Route별 테스트

각 엔드포인트마다 아래 3가지는 공통 검증한다.

1. 경로 매칭이 올바른 private handler로 분기되는지
2. 요청 body/query가 DTO 변환 전 필수값 검증을 통과하는지
3. 응답 body가 `success/data/meta.requestId` 구조를 유지하는지

#### 예시

- `GET /api/admin/owners` -> `adminRepository.findOwners(...)` 호출 여부
- `POST /api/admin/owners` -> 기본 그룹 생성 flag 처리 여부
- `POST /api/admin/drivers/{driverId}/create-linked-account` -> `cognitoUserManager.inviteUser(...)` 후 repository 저장 여부
- `POST /api/admin/verifications/run` -> `accepted_count` 응답 매핑 여부
- `POST /api/admin/app-activations/{activationId}/resend-invite` -> Cognito 재초대 실패 시 오류 응답 여부

### 8.4 Repository 테스트 구조

`AuroraAdminRepositoryTest`는 두 층으로 나눈다.

- 기존 위임 메서드 테스트: 기존 `AuroraRepository` public method 호출 위임 여부
- 신규 named-parameter SQL 테스트: `executeQueryNew`, `executeUpdateNew`, `executeCountNew` 사용 여부와 결과 매핑

---

## 9. 스펙과 현재 코드 사이의 주의점

### 9.1 경로 표현 차이

- 스펙 기본 경로: `/api/admin`
- 실제 API Gateway URL: `https://a1xpskzj48.execute-api.ap-northeast-2.amazonaws.com/v1/api/admin`
- 현재 `ApiAdapterHandler`는 `/v1` prefix를 제거한다.

따라서 `AdminHandler.normalizePath(...)`는 `/v1/admin/*`와 `/api/admin/*` 모두 허용해야 한다.

### 9.2 스펙에 권한이 명시되지 않은 엔드포인트

아래 영역은 개별 권한 라인이 없다.

- Vehicle 일부
- Driver 전체
- Account Linking 전체
- Mapping 전체
- Verification 전체
- App Activation 전체

설계 기준은 문서 1.2의 공통 규칙(`Installer, Operator, Admin`)을 기본값으로 사용한다.

### 9.3 스펙 응답 본문이 비어 있는 엔드포인트

아래 엔드포인트는 요청 예시는 있으나 응답 예시가 없다.

- `PUT /api/admin/groups/{groupId}`
- `PUT /api/admin/devices/{deviceId}`
- `POST /api/admin/devices/check-duplicate`
- `PUT /api/admin/vehicles/{vehicleId}`
- `POST /api/admin/drivers`
- `PUT /api/admin/drivers/{driverId}`

이 문서에서는 기존 handler 패턴에 맞춰 아래처럼 설계한다.

- update 계열 -> `{resource_id}` 반환
- duplicate check -> `duplicated`, `duplicated_fields`

### 9.4 응답 envelope

현재 개별 admin handler들은 `meta.timestamp`만 넣고 `requestId`를 넣지 않는다. 반면 `MobileHandler`는 `meta.requestId`와 `X-Request-Id` 헤더를 모두 넣는다.

AdminHandler 설계는 MobileHandler에 맞춰 아래 구조로 통일한다.

```json
{
  "success": true,
  "data": { },
  "meta": {
    "timestamp": "...",
    "requestId": "req_xxx"
  }
}
```

---

## 10. 최종 권장안

가장 변경 폭이 적고 현재 코드와 잘 맞는 방향은 다음이다.

1. `ApiAdapterHandler`는 Lambda entrypoint로 유지한다.
2. Admin 경로는 `AdminHandler` 단일 진입점으로 위임한다.
3. `AdminHandler`는 `MobileHandler`와 동일한 인증/분기/공통응답 패턴을 따른다.
4. 기존 `AuroraRepository` 메서드가 있는 영역은 위임한다.
5. 신규 영역은 `AdminRepository` + `AuroraAdminRepository`로 추가하고, SQL은 `executeQueryNew` 계열을 사용한다.
6. Cognito 관련 엔드포인트는 `CognitoUserManager`를 조합한다.

이 구조면 기존 코드 스타일을 유지하면서도, 누락된 37개 admin endpoint를 하나의 핸들러 설계 아래에 정리할 수 있다.
