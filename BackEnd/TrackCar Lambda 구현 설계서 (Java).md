# TrackCar Lambda 구현 설계서 (Java)

- 작성일: 2026-03-28
- 버전: v3.3
- 상태: 초안
- 기반: TrackCar 플랫폼 아키텍처 상세 설계 v3.0.1
- **구현 언어: Java 25 + AWS Lambda RequestHandler 패턴**

---

## 1. 문서 개요

### 1.1 목적

본 문서는 TrackCar 백엔드의 AWS Lambda 함수들의 상세 구현 설계를 정의한다. 각 함수의 트리거, 입력/출력 스키마, 처리 로직, 에러 처리, 의존 관계를 명시하여 개발 및 유지보수의 기준을 제공한다.

### 1.2 기술 스택

| 항목 | 버전/라이브러리 |
|------|----------------|
| Java | 25 |
| AWS Lambda Runtime | java25 |
| AWS SDK for Java v2 | 2.29.x |
| Jackson | 2.17.x |
| Lombok | 1.18.44 |
| Logback | 1.5.8 |
| Aurora 연동 | AWS RDS Data API (HTTP) |

### 1.3 핵심 설계 결정

1. **Spring Boot 미사용**: Lambda Cold Start 최적화를 위해 Spring Boot, Spring Cloud Function 제거
2. **RequestHandler 패턴**: 모든 Lambda 함수는 `RequestHandler` 인터페이스 직접 구현
3. **RDS Data API**: Aurora 연동 시 JDBC 드라이버 대신 AWS RDS Data API (HTTP) 사용
4. **순수 Java 의존성**: AWS SDK v2와 Jackson만 사용

### 1.4 Gradle 의존성

```groovy
// build.gradle
plugins {
    id 'java'
    id 'io.spring.dependency-management' version '1.1.7'
}

java {
    sourceCompatibility = JavaVersion.VERSION_25
    targetCompatibility = JavaVersion.VERSION_25
}

repositories {
    mavenCentral()
}

ext {
    set('awsSdkVersion', "2.29.0")
    set('jacksonVersion', "2.17.2")
    set('lombokVersion', "1.18.44")
    set('logbackVersion', "1.5.8")
    set('jose4jVersion', "0.9.6")
    set('rdsDataApiClientVersion', "2.0.0")
}

dependencyManagement {
    imports {
        mavenBom "software.amazon.awssdk:bom:${awsSdkVersion}"
    }
}

dependencies {
    // AWS SDK v2
    implementation 'software.amazon.awssdk:lambda'
    implementation 'software.amazon.awssdk:dynamodb'
    implementation 'software.amazon.awssdk:rdsdata'
    implementation 'software.amazon.awssdk:s3'
    implementation 'software.amazon.awssdk:sqs'
    implementation 'software.amazon.awssdk:iot'
    implementation 'software.amazon.awssdk:secretsmanager'
    implementation 'software.amazon.awssdk:cloudwatch'
    implementation 'software.amazon.awssdk:cognitoidentityprovider'
    
    // AWS Lambda Events (SQS, EventBridge, API Gateway)
    implementation 'com.amazonaws:aws-lambda-java-events:3.1.0'
    implementation 'com.amazonaws:aws-lambda-java-core:1.2.3'
    
    // JSON (pure Jackson, no Spring)
    implementation "com.fasterxml.jackson.core:jackson-databind:${jacksonVersion}"
    implementation "com.fasterxml.jackson.datatype:jackson-datatype-jsr310:${jacksonVersion}"
    
    // JWT (for Cognito authentication)
    implementation "org.bitbucket.b_c:jose4j:${jose4jVersion}"
    
    // RDS Data API Client Library (named parameter binding)
    implementation "software.amazon.rdsdata:rds-data-api-client-library-java:${rdsDataApiClientVersion}"
    
    // Lombok
    compileOnly "org.projectlombok:lombok:${lombokVersion}"
    annotationProcessor "org.projectlombok:lombok:${lombokVersion}"
    
    // Logging
    runtimeOnly "ch.qos.logback:logback-classic:${logbackVersion}"
    runtimeOnly 'org.slf4j:slf4j-api'
    
    // Testing
    testImplementation 'org.mockito:mockito-core'
}
```

---

## 2. 프로젝트 구조

```
aws-functions/
├── build.gradle                                    # Spring 제거, JDK 25
├── src/main/java/com/trackcar/
│   ├── ingest/                                    # 수집 파이프라인
│   │   ├── IngestAuthValidateHandler.java        # SQS 트리거
│   │   ├── TelemetryProcessHandler.java          # SQS 트리거
│   │   └── PurgeOldDataHandler.java              # EventBridge 트리거
│   ├── etas/                                     # eTAS 배치
│   │   ├── FetchTargetsHandler.java              # EventBridge 트리거
│   │   ├── FormatterHandler.java                 # SQS 트리거
│   │   └── TransmitterHandler.java               # SQS 트리거
│   ├── api/                                      # API Adapter
│   │   └── ApiAdapterHandler.java                # API Gateway 트리거
│   ├── common/                                   # 공통
│   │   ├── context/
│   │   │   ├── MobileRequestContext.java        # Mobile API 요청 context
│   │   │   └── AdminRequestContext.java         # Admin API 요청 context
│   │   ├── model/
│   │   │   ├── TelemetryPayload.java
│   │   │   ├── CleanStreamRecord.java
│   │   │   ├── ETASMessage.java
│   │   │   └── ApiResponse.java
│   │   ├── repository/
│   │   │   ├── AuroraRepository.java             # RDS Data API
│   │   │   └── DynamoDbRepository.java
│   │   ├── client/
│   │   │   └── SqsClient.java
│   │   ├── exception/
│   │   │   ├── ValidationException.java
│   │   │   └── InternalException.java
│   │   └── util/
│   │       ├── CognitoAuthValidator.java         # JWT 검증
│   │       ├── CognitoUserManager.java          # Cognito Admin API
│   │       ├── MetricsLogger.java
│   │       └── JsonUtils.java
│   ├── api/                                      # REST API
│   │   └── handlers/
│   │       ├── AdminHandler.java                # Installer Admin API Handler
│   │       └── MobileHandler.java               # Mobile API Handler
├── src/main/resources/
│   └── logback.xml
└── src/test/java/
```

**주요 변경사항:**
- `Function` suffix → `Handler` suffix로 변경
- Spring `@Configuration`, `@Bean`, `@Repository` 제거
- `Application.java` (Spring Boot 진입점) 제거
- `application.yml` 제거

---

## 3. Handler 패턴 구현 예시

### 3.1 기본 Handler 구조

```java
// FetchTargetsHandler.java
package com.trackcar.etas;

import com.amazonaws.services.lambda.runtime.Context;
import com.amazonaws.services.lambda.runtime.RequestHandler;
import com.amazonaws.services.lambda.runtime.events.EventBridgeEvent;
import com.trackcar.common.repository.AuroraRepository;
import com.trackcar.common.util.MetricsLogger;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import java.util.List;
import java.util.Map;

public class FetchTargetsHandler implements RequestHandler<EventBridgeEvent, String> {
    
    private static final Logger log = LoggerFactory.getLogger(FetchTargetsHandler.class);
    
    // 직접 초기화 (Spring 없음)
    private final AuroraRepository auroraRepository;
    private final MetricsLogger metricsLogger;
    
    public FetchTargetsHandler() {
        this.auroraRepository = new AuroraRepository();
        this.metricsLogger = new MetricsLogger();
    }
    
    @Override
    public String handleRequest(EventBridgeEvent event, Context context) {
        log.info("Starting eTAS target fetch");
        
        try {
            // 환경변수 직접 참조 (Spring @Value 없음)
            String clusterArn = System.getenv("AURORA_CLUSTER_ARN");
            String secretArn = System.getenv("AURORA_SECRET_ARN");
            
            // 비지니스 로직
            List<Map<String, Object>> targets = auroraRepository.findETASTargetVehicles();
            
            // 결과 반환
            log.info("Found {} eTAS targets", targets.size());
            return "SUCCESS: " + targets.size() + " targets processed";
            
        } catch (Exception e) {
            log.error("Error processing eTAS targets", e);
            return "ERROR: " + e.getMessage();
        }
    }
}
```

### 3.2 SQS 트리거 Handler

```java
// FormatterHandler.java
package com.trackcar.etas;

import com.amazonaws.services.lambda.runtime.Context;
import com.amazonaws.services.lambda.runtime.RequestHandler;
import com.amazonaws.services.lambda.runtime.events.SQSEvent;
import com.trackcar.common.repository.AuroraRepository;
import com.trackcar.common.util.JsonUtils;
import com.trackcar.common.util.MetricsLogger;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import java.util.List;

public class FormatterHandler implements RequestHandler<SQSEvent, String> {
    
    private static final Logger log = LoggerFactory.getLogger(FormatterHandler.class);
    
    private final AuroraRepository auroraRepository;
    private final S3Client s3Client;
    private final SqsClient sqsClient;
    private final JsonUtils jsonUtils;
    private final MetricsLogger metricsLogger;
    
    public FormatterHandler() {
        this.auroraRepository = new AuroraRepository();
        this.s3Client = new S3Client();
        this.sqsClient = new SqsClient();
        this.jsonUtils = new JsonUtils();
        this.metricsLogger = new MetricsLogger();
    }
    
    @Override
    public String handleRequest(SQSEvent event, Context context) {
        log.info("Processing {} SQS messages", event.getRecords().size());
        
        int successCount = 0;
        int errorCount = 0;
        
        for (SQSEvent.SQSMessage record : event.getRecords()) {
            try {
                // 메시지 처리
                processMessage(record);
                successCount++;
                
            } catch (Exception e) {
                log.error("Error processing message: {}", record.getMessageId(), e);
                errorCount++;
            }
        }
        
        metricsLogger.emitCount("formatter_records_processed", 
            Map.of("success", String.valueOf(successCount), "error", String.valueOf(errorCount)));
        
        return "Processed: " + successCount + " success, " + errorCount + " errors";
    }
    
    private void processMessage(SQSEvent.SQSMessage record) throws Exception {
        // S3에서 Parquet 파일 읽기
        // 별표 2 규격 TXT 변환
        // S3에 TXT 업로드
        // etas_transmission_target 상태 업데이트
    }
}
```

### 3.3 API Gateway Admin Handler 패턴

`api-adapter-client` Lambda는 API Gateway 요청을 받아 경로 기준으로 Admin API와 Mobile API로 위임합니다.

```java
// ApiAdapterHandler.java
public class ApiAdapterHandler implements RequestHandler<APIGatewayProxyRequestEvent, APIGatewayProxyResponseEvent> {

    private final AdminHandler adminHandler;
    private final MobileHandler mobileHandler;

    @Override
    public APIGatewayProxyResponseEvent handleRequest(APIGatewayProxyRequestEvent event, Context context) {
        String path = event.getPath();
        String method = event.getHttpMethod();
        Map<String, String> queryParams = event.getQueryStringParameters();
        Map<String, Object> body = parseBody(event.getBody());
        String normalizedPath = normalizePath(path);

        if (normalizedPath.startsWith("/api/admin/") || normalizedPath.startsWith("/v1/admin/")) {
            String authHeader = event.getHeaders().get("Authorization");
            return adminHandler.handleRequest(path, method, queryParams, body, authHeader, objectMapper, requestId);
        }

        if (normalizedPath.startsWith("/v1/mobile/") || normalizedPath.startsWith("/mobile/")) {
            String authHeader = event.getHeaders().get("Authorization");
            return mobileHandler.handleRequest(path, method, queryParams, body, authHeader, objectMapper);
        }

        return notFound(objectMapper, requestId);
    }
}
```

`AdminHandler`의 공통 흐름은 다음과 같습니다.

1. `Authorization` 헤더에서 Bearer 토큰 추출
2. `CognitoAuthValidator.validateToken(...)` 호출
3. JWT claims를 `AdminRequestContext`로 변환
4. HTTP Method별 분기 (`GET`, `POST`, `PUT`, `DELETE`)
5. 엔드포인트별 private 메서드 호출
6. 공통 JSON 응답과 `X-Request-Id` 헤더 반환

이 구조로 Admin API의 인증, 권한 검사, 경로 분기, 공통 응답 포맷을 `AdminHandler`에 일원화합니다.

---

## 4. 공통 모듈

### 4.1 AuroraRepository (RDS Data API)

```java
// AuroraRepository.java
package com.trackcar.common.repository;

import software.amazon.awssdk.services.rdsdata.RdsDataClient;
import software.amazon.awssdk.services.rdsdata.model.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import java.util.*;

/**
 * Aurora PostgreSQL Data API를 사용한 Repository
 * Spring JDBC 미사용 - AWS RDS Data API 직접 호출
 */
public class AuroraRepository {
    
    private static final Logger log = LoggerFactory.getLogger(AuroraRepository.class);
    
    private final RdsDataClient rdsDataClient;
    private final String clusterArn;
    private final String secretArn;
    private final String database;
    
    public AuroraRepository() {
        this.rdsDataClient = RdsDataClient.builder()
            .region(software.amazon.awssdk.regions.Region.AP_NORTHEAST_2)
            .build();
        
        // 환경변수 직접 참조
        this.clusterArn = System.getenv("AURORA_CLUSTER_ARN");
        this.secretArn = System.getenv("AURORA_SECRET_ARN");
        this.database = "trackcar";
    }
    
    /**
     * 차량-DTG 바인딩 조회
     */
    public Map<String, Object> findVehicleBinding(String deviceId) {
        String sql = """
            SELECT vdb.binding_id, vdb.vehicle_id, vdb.interface_id, vdb.interface_version,
                   v.vehicle_no, v.vin, v.vehicle_type, v.organization_id,
                   o.transport_biz_no
            FROM vehicle_device_binding vdb
            JOIN vehicle v ON vdb.vehicle_id = v.vehicle_id
            JOIN device d ON vdb.device_id = d.device_id
            JOIN organization o ON v.organization_id = o.organization_id
            WHERE (vdb.device_id = $1 OR d.serial_no = $1)
            AND vdb.status = 'ACTIVE'
            AND v.deleted_at IS NULL
            """;
        
        ExecuteStatementRequest request = ExecuteStatementRequest.builder()
            .resourceArn(clusterArn)
            .secretArn(secretArn)
            .database(database)
            .sql(sql)
            .parameters(Parameter.builder()
                .name("deviceId")
                .value(Uri.builder().stringValue(deviceId).build())
                .build())
            .build();
        
        ExecuteStatementResponse response = rdsDataClient.executeStatement(request);
        
        if (response.records().isEmpty()) {
            return null;
        }
        
        return mapToMap(response.records().get(0), 
            List.of("binding_id", "vehicle_id", "interface_id", "interface_version",
                   "vehicle_no", "vin", "vehicle_type", "organization_id", "transport_biz_no"));
    }
    
    /**
     * eTAS 전송 대상 조회
     */
    public List<Map<String, Object>> findETASTargetVehicles() {
        String sql = """
            SELECT v.vehicle_id, v.vehicle_no, v.vin, v.vehicle_type,
                   o.transport_biz_no, d.driver_code,
                   o.organization_id
            FROM vehicle v
            JOIN organization o ON v.organization_id = o.organization_id
            LEFT JOIN vehicle_driver_link vdl ON v.vehicle_id = vdl.vehicle_id AND vdl.status = 'ACTIVE'
            LEFT JOIN driver d ON vdl.driver_id = d.driver_id
            WHERE v.vehicle_status = 'ACTIVE'
            AND v.install_status = 'VERIFIED'
            AND v.deleted_at IS NULL
            AND o.status = 'ACTIVE'
            """;
        
        ExecuteStatementRequest request = ExecuteStatementRequest.builder()
            .resourceArn(clusterArn)
            .secretArn(secretArn)
            .database(database)
            .sql(sql)
            .build();
        
        ExecuteStatementResponse response = rdsDataClient.executeStatement(request);
        
        return response.records().stream()
            .map(record -> mapToMap(record, 
                List.of("vehicle_id", "vehicle_no", "vin", "vehicle_type",
                       "transport_biz_no", "driver_code", "organization_id")))
            .toList();
    }
    
    /**
     * Trip 저장
     */
    public void saveTrip(String tripId, String vehicleId, String driverId,
                        String startedAt, String status, String tripDate) {
        String sql = """
            INSERT INTO trip_meta 
            (trip_id, vehicle_id, driver_id, started_at, status, trip_date, created_at, updated_at)
            VALUES ($1, $2, $3, $4, $5, $6, CURRENT_TIMESTAMP, CURRENT_TIMESTAMP)
            ON CONFLICT (trip_id) DO UPDATE SET
                ended_at = EXCLUDED.ended_at,
                distance_km = EXCLUDED.distance_km,
                duration_min = EXCLUDED.duration_min,
                status = EXCLUDED.status,
                updated_at = CURRENT_TIMESTAMP
            """;
        
        ExecuteStatementRequest request = ExecuteStatementRequest.builder()
            .resourceArn(clusterArn)
            .secretArn(secretArn)
            .database(database)
            .sql(sql)
            .parameters(
                Parameter.builder().name("tripId").value(Uri.builder().stringValue(tripId).build()).build(),
                Parameter.builder().name("vehicleId").value(Uri.builder().stringValue(vehicleId).build()).build(),
                Parameter.builder().name("driverId").value(Uri.builder().stringValue(driverId).build()).build(),
                Parameter.builder().name("startedAt").value(Uri.builder().stringValue(startedAt).build()).build(),
                Parameter.builder().name("status").value(Uri.builder().stringValue(status).build()).build(),
                Parameter.builder().name("tripDate").value(Uri.builder().stringValue(tripDate).build()).build()
            )
            .build();
        
        rdsDataClient.executeStatement(request);
    }
    
    /**
     * eTAS 전송 대상 상태 업데이트
     */
    public void updateETASTargetStatus(String targetId, String status, String fileName, String errorCode, String errorMessage) {
        String sql = """
            UPDATE etas_transmission_target
            SET status = $2,
                generated_file_name = $3,
                transmitted_at = CASE WHEN $2 = 'COMPLETED' THEN CURRENT_TIMESTAMP ELSE transmitted_at END,
                error_code = $4,
                error_message = $5,
                attempt_count = attempt_count + 1,
                updated_at = CURRENT_TIMESTAMP
            WHERE target_id = $1
            """;
        
        ExecuteStatementRequest request = ExecuteStatementRequest.builder()
            .resourceArn(clusterArn)
            .secretArn(secretArn)
            .database(database)
            .sql(sql)
            .parameters(
                Parameter.builder().name("targetId").value(Uri.builder().stringValue(targetId).build()).build(),
                Parameter.builder().name("status").value(Uri.builder().stringValue(status).build()).build(),
                Parameter.builder().name("fileName").value(Uri.builder().stringValue(fileName != null ? fileName : "").build()).build(),
                Parameter.builder().name("errorCode").value(Uri.builder().stringValue(errorCode != null ? errorCode : "").build()).build(),
                Parameter.builder().name("errorMessage").value(Uri.builder().stringValue(errorMessage != null ? errorMessage : "").build()).build()
            )
            .build();
        
        rdsDataClient.executeStatement(request);
    }
    
    private Map<String, Object> mapToMap(List<Field> fields, List<String> keys) {
        Map<String, Object> result = new HashMap<>();
        for (int i = 0; i < keys.size() && i < fields.size(); i++) {
            Field field = fields.get(i);
            String key = keys.get(i);
            
            if (field.stringValue() != null) {
                result.put(key, field.stringValue());
            } else if (field.longValue() != null) {
                result.put(key, field.longValue());
            } else if (field.doubleValue() != null) {
                result.put(key, field.doubleValue());
            } else if (field.booleanValue() != null) {
                result.put(key, field.booleanValue());
            }
        }
        return result;
    }
}
```

현재 `AuroraRepository`는 수집/eTAS 처리뿐 아니라 Admin API용 조회/생성/수정 책임도 함께 담당합니다.

- Admin 목록/상세/수정 메서드: `findOwners`, `findGroups`, `findDevices`, `findVehicles`, `findDrivers`
- Admin 부가 기능 메서드: `findAccountCandidates`, `findVerifications`, `findAppActivations`, `findSystemUsers`
- 매핑/상태 변경 메서드: `createBundleMapping`, `deleteDeviceVehicleMapping`, `deleteVehicleDriverMapping`, `markAppActivationLive`
- 공통 결과 래퍼: `AdminQueryResult`

또한 Named parameter 기반 헬퍼 메서드가 추가되어, 신규 Admin API 구현에서도 SQL 가독성을 높일 수 있습니다.

- `executeQueryNew(...)`
- `executeUpdateNew(...)`
- `executeCountNew(...)`

### 4.2 DynamoDbRepository

```java
// DynamoDbRepository.java
package com.trackcar.common.repository;

import software.amazon.awssdk.services.dynamodb.DynamoDbClient;
import software.amazon.awssdk.services.dynamodb.model.*;
import java.time.Instant;
import java.util.HashMap;
import java.util.Map;

/**
 * DynamoDB Repository - Spring 미사용
 */
public class DynamoDbRepository {
    
    private final DynamoDbClient dynamoDbClient;
    
    public DynamoDbRepository() {
        this.dynamoDbClient = DynamoDbClient.builder()
            .region(software.amazon.awssdk.regions.Region.AP_NORTHEAST_2)
            .build();
    }
    
    /**
     * 멱등성 체크 및 설정
     * @return true if duplicate (already exists), false if new
     */
    public boolean checkAndSetDedup(String tableName, String key) {
        String pk = "KEY#" + key;
        
        // 존재 여부 확인
        GetItemRequest getRequest = GetItemRequest.builder()
            .tableName(tableName)
            .key(Map.of(
                "PK", AttributeValue.builder().s(pk).build(),
                "SK", AttributeValue.builder().s("DEDUP").build()
            ))
            .consistentRead(true)
            .build();
        
        GetItemResponse getResponse = dynamoDbClient.getItem(getRequest);
        
        if (getResponse.hasItem() && !getResponse.item().isEmpty()) {
            return true; // 이미 존재 = 중복
        }
        
        // 존재하지 않으면 저장 (TTL 7일)
        long ttl = Instant.now().getEpochSecond() + 604800;
        
        Map<String, AttributeValue> item = new HashMap<>();
        item.put("PK", AttributeValue.builder().s(pk).build());
        item.put("SK", AttributeValue.builder().s("DEDUP").build());
        item.put("processedAt", AttributeValue.builder().s(Instant.now().toString()).build());
        item.put("status", AttributeValue.builder().s("PROCESSED").build());
        item.put("TTL", AttributeValue.builder().n(String.valueOf(ttl)).build());
        
        PutItemRequest putRequest = PutItemRequest.builder()
            .tableName(tableName)
            .item(item)
            .conditionExpression("attribute_not_exists(PK)")
            .build();
        
        try {
            dynamoDbClient.putItem(putRequest);
            return false; // 새로 저장됨 = 중복 아님
        } catch (ConditionalCheckFailedException e) {
            return true; // 조건 충족 안됨 = 이미 존재 = 중복
        }
    }
    
    /**
     * 차량 마지막 상태 조회
     */
    public Map<String, AttributeValue> getLastState(String tableName, String vehicleId) {
        GetItemRequest request = GetItemRequest.builder()
            .tableName(tableName)
            .key(Map.of(
                "PK", AttributeValue.builder().s("VEHICLE#" + vehicleId).build(),
                "SK", AttributeValue.builder().s("STATE").build()
            ))
            .consistentRead(true)
            .build();
        
        GetItemResponse response = dynamoDbClient.getItem(request);
        return response.hasItem() ? response.item() : null;
    }
    
    /**
     * 배치 쓰기
     */
    public BatchWriteItemResponse batchWrite(BatchWriteItemRequest request) {
        return dynamoDbClient.batchWriteItem(request);
    }
}
```

### 4.3 MetricsLogger

```java
// MetricsLogger.java
package com.trackcar.common.util;

import software.amazon.awssdk.services.cloudwatch.model.*;
import software.amazon.awssdk.services.cloudwatch.CloudWatchClient;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import java.time.Instant;
import java.util.Map;

/**
 * CloudWatch 메트릭 로깅 - Spring 미사용
 */
public class MetricsLogger {
    
    private static final Logger log = LoggerFactory.getLogger(MetricsLogger.class);
    private static final String NAMESPACE = "TrackCar";
    
    private final CloudWatchClient cloudWatchClient;
    private final String functionName;
    
    public MetricsLogger() {
        this.cloudWatchClient = CloudWatchClient.builder()
            .region(software.amazon.awssdk.regions.Region.AP_NORTHEAST_2)
            .build();
        this.functionName = System.getenv().getOrDefault("AWS_LAMBDA_FUNCTION_NAME", "unknown");
    }
    
    public void emitCount(String metricName, Map<String, String> dimensions) {
        try {
            MetricDatum datum = MetricDatum.builder()
                .metricName(metricName)
                .unit(StandardUnit.COUNT)
                .value(1.0)
                .timestamp(Instant.now())
                .dimensions(dimensions.entrySet().stream()
                    .map(e -> Dimension.builder()
                        .name(e.getKey())
                        .value(e.getValue())
                        .build())
                    .toList())
                .build();
            
            cloudWatchClient.putMetricData(PutMetricDataRequest.builder()
                .namespace(NAMESPACE)
                .metricData(datum)
                .build());
                
        } catch (Exception e) {
            log.warn("Failed to emit metric: {}", e.getMessage());
        }
    }
    
    public void emitCount(String metricName) {
        emitCount(metricName, Map.of());
    }
}
```

### 4.4 AdminRequestContext

`AdminRequestContext`는 Admin API 요청 처리 시 JWT claims를 기반으로 생성되는 요청 컨텍스트입니다.

```java
@Data
@Builder
public class AdminRequestContext {
    private final String userId;
    private final String username;
    private final String name;
    private final String email;
    private final String role;
    private final List<String> groups;
    private final String requestId;

    public boolean isInstaller();
    public boolean isOperator();
    public boolean isAdmin();
    public boolean isOperatorOrAdmin();
}
```

이 컨텍스트는 `AdminHandler`에서 인증 성공 직후 생성되며, 엔드포인트별 권한 검사와 감사 추적용 `requestId` 전달에 사용됩니다.

---

## 5. Lambda 함수 매핑

| 함수명 | Handler | 런타임 | 트리거 | 목적 |
|--------|---------|--------|--------|------|
| ingest-auth-validate | `IngestAuthValidateHandler` | java25 | SQS (raw-ingest) | 인증/검증, 중복 방지 |
| telemetry-process | `TelemetryProcessHandler` | java25 | SQS (clean-queue) | DynamoDB + Aurora 저장 |
| etas-fetch-targets | `FetchTargetsHandler` | java25 | EventBridge (일배치) | eTAS 전송 대상 조회 |
| etas-formatter | `FormatterHandler` | java25 | SQS (etas-processing) | 파일 생성, S3 업로드 |
| etas-transmitter | `TransmitterHandler` | java25 | SQS (etas-transmit) | eTAS API 전송 |
| api-adapter-client | `ApiAdapterHandler` | java25 | API Gateway | REST API 요청 처리 |
| purge-old-data | `PurgeOldDataHandler` | java25 | EventBridge (02:00 KST) | 3년 된 데이터 자동 삭제 |

`api-adapter-client`는 단일 Lambda 진입점이지만, 내부에서는 다음과 같이 하위 핸들러로 위임합니다.

- `ApiAdapterHandler -> AdminHandler`: Installer Admin Web용 Admin API 처리
- `ApiAdapterHandler -> MobileHandler`: Mobile App용 Mobile API 처리

Admin API는 `/api/admin/*`, `/v1/admin/*` 경로를 사용하며, 주요 기능은 다음 범위를 포함합니다.

- Dashboard
- Owners / Groups / Devices / Vehicles / Drivers / Members
- Account linking / Quick Mapping
- Verifications
- App Activations
- System Users

---

## 6. 환경변수 참조

Lambda 함수에서 환경변수는 `System.getenv()`로 직접 참조합니다:

```java
// Aurora 연결
String clusterArn = System.getenv("AURORA_CLUSTER_ARN");
String secretArn = System.getenv("AURORA_SECRET_ARN");

// DynamoDB 연결
String tableName = System.getenv("VEHICLE_STATE_TABLE_NAME");

// SQS 연결
String queueUrl = System.getenv("CLEAN_QUEUE_URL");

// S3 연결
String bucketName = System.getenv("RAW_TELEMETRY_BUCKET");

// eTAS API
String etasApiUrl = System.getenv("ETAS_API_URL");
String etasApiKey = System.getenv("ETAS_API_KEY");

// Cognito 연결
String cognitoUserPoolId = System.getenv("COGNITO_USER_POOL_ID");
String cognitoClientId = System.getenv("COGNITO_CLIENT_ID");
```

Admin API 구현에서도 위 환경변수를 동일하게 사용합니다.

- `COGNITO_USER_POOL_ID`, `COGNITO_CLIENT_ID`: Admin API JWT 검증
- `AURORA_CLUSTER_ARN`, `AURORA_SECRET_ARN`, `AURORA_DATABASE`: Admin API 조회/수정 SQL 실행
- `AWS_REGION`: Cognito JWKS 조회 및 Aurora/Data API 클라이언트 리전 설정

---

## 7. Cognito 연동

### 7.1 CognitoAuthValidator (JWT 검증)

Cognito에서 발급된 JWT 토큰을 검증합니다:

```java
// CognitoAuthValidator.java
public class CognitoAuthValidator {
    
    public AuthResult validateToken(String token) {
        // JWKS fetching and caching
        // JWT signature verification
        // Audience validation
        // Claims extraction (sub, email, cognito:groups, custom:*)
    }
}
```

`CognitoAuthValidator`는 현재 `AdminHandler`와 `MobileHandler` 양쪽에서 공통으로 사용합니다. Admin API에서는 검증 성공 후 claims를 `AdminRequestContext`로 변환해 역할 기반 권한 검사에 활용합니다.

### 7.2 CognitoUserManager (Cognito Admin API)

Cognito User Pool 사용자를 관리합니다:

```java
// CognitoUserManager.java
public class CognitoUserManager {
    
    public InviteResult inviteUser(String email, String name, 
                                  String temporaryPassword, 
                                  List<String> groups,
                                  Map<String, String> customAttributes) {
        // admin_create_user 호출
        // 임시 비밀번호 발송
        // 그룹 할당
    }
    
    public boolean disableUser(String username);
    public boolean enableUser(String username);
    public boolean deleteUser(String username);
    public boolean resendInvitation(String email);
}
```

`CognitoUserManager`는 Admin API의 사용자 초대/관리 흐름에서 사용됩니다. 특히 앱 활성화 초대, 시스템 사용자 생성/초대 같은 관리 기능에서 Cognito Admin API 연동 지점으로 사용합니다.

현재 `system/users`의 source of truth는 실제 DB가 아니라 Cognito User Pool이다.

- `POST /api/admin/system/users`
  - Cognito `AdminCreateUser` 기반으로 사용자 생성
  - 임시 비밀번호 발급
  - 응답에 `login_username`, `temporary_password`, `password_change_required=true` 포함
- `GET /api/admin/system/users`
  - Cognito 사용자와 Cognito group 기준으로 목록 구성
- 최초 로그인
  - 임시 비밀번호 로그인 시 `NEW_PASSWORD_REQUIRED`
  - 프론트가 `completeNewPasswordChallenge`로 새 비밀번호 설정

운영 전제:
- User Pool group: `Admin`, `Operator`, `Installer`
- 관리자 테스트 계정이 `Admin` group 소속일 것
- Lambda 실행 역할에 Cognito Admin API 권한이 있을 것

---

## 8. RDS Data API Client Library

### 8.1 개요

[RDS Data API Client Library](https://github.com/awslabs/rds-data-api-client-library-java)를 사용하면 Named parameter 바인딩이 가능합니다.

```groovy
// build.gradle
implementation "software.amazon.rdsdata:rds-data-api-client-library-java:2.0.0"
```

### 8.2 헬퍼 메서드 사용법

```java
// Named parameter 바인딩
List<Map<String, Object>> results = executeQueryNew(
    "SELECT * FROM table WHERE id = :id AND status = :status",
    "id", vehicleId,
    "status", "ACTIVE"
);

// INSERT/UPDATE
executeUpdateNew(
    "INSERT INTO table (name, value) VALUES (:name, :value)",
    "name", name,
    "value", value
);

// COUNT
long count = executeCountNew(
    "SELECT COUNT(*) FROM table WHERE status = :status",
    "status", "ACTIVE"
);
```

### 8.3 장점

| 항목 | 기존 SDK | Client Library |
|------|---------|---------------|
| 파라미터 문법 | `$1`, `$2` positional | `:name` named |
| 가독성 | 낮음 | 높음 |
| SQL 가독성 | 나쁨 | 좋음 |

---

## 9. 변경 이력 (Changelog)

- **v3.3 (2026-03-28):**
  - `AdminHandler` 추가 (Installer Admin API 통합 처리)
  - `AdminRequestContext` 추가 (Admin JWT claims 기반 요청 컨텍스트)
  - `ApiAdapterHandler`가 Admin API와 Mobile API를 하위 핸들러로 위임하도록 정리
  - `AuroraRepository`에 Admin API용 목록/상세/상태변경 메서드 추가
  - 메인 Lambda 설계서에 Admin API 처리 구조 반영

- **v3.2 (2026-03-28):**
  - RDS Data API Client Library 추가 (`software.amazon.rdsdata:rds-data-api-client-library-java:2.0.0`)
  - AuroraRepository에 Named parameter 헬퍼 메서드 추가
  - JAR 크기: 31MB → 33MB

- **v3.1 (2026-03-28):**
  - CognitoUserManager 추가 (Cognito Admin API 래퍼)
  - CognitoAuthValidator 추가 (JWT 검증)
  - jose4j 라이브러리 추가
  - cognitoidentityprovider SDK 추가
  - MobileRequestContext 추가
  - MobileHandler 추가 (Mobile API 처리)

- **v3.0 (2026-03-27):**
  - Java 25로 업그레이드
  - Spring Boot, Spring Cloud Function 완전 제거
  - Handler 패턴 도입 (`Function` → `Handler`)
  - `RequestHandler` 인터페이스 직접 구현
  - RDS Data API 기반 Aurora 연동 (JDBC 미사용)
  - 환경변수 직접 참조 (`System.getenv()`)
  - `AuroraRepository.java` - Data API 파라미터 인덱스 `$1`, `$2` 형식 사용

- **v2.0 (2026-03-25):**
  - Java 17 + Spring Boot 3.2.x 적용
  - Spring Cloud Function Adapter 도입
  - RDS Proxy JDBC 연결

- **v1.0 (2026-03-25):**
  - 최초 구현 설계서 작성

---

## 10. 참고 문서

| 문서명 | 버전 | 요약 |
|--------|------|------|
| TrackCar Aurora PostgreSQL 물리 스키마 명세서 | v3.1 | Aurora 테이블 정의 |
| RDS Data API Client Library | 2.0.0 | Named parameter 바인딩 |
| TrackCar AdminHandler 구현 설계 | - | Installer Admin API 상세 설계 |
| TrackCar DynamoDB 물리 스키마 명세서 | v2.0 | DynamoDB 테이블 정의 |
| TrackCar 플랫폼 아키텍처 상세 설계 | v3.0.1 | 전체 시스템 아키텍처 |
| AWS Lambda Developer Guide | - | Lambda 기본 개념 |
| AWS RDS Data API | - | Aurora Data API |
