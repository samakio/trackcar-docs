# TrackCar Mobile App - Replit 개발 가이드

## 1. 프로젝트 개요

**프로젝트명**: TrackCar Mobile App
**플랫폼**: Android/iOS (React Native 또는 PWA)
**목적**: 운전자/관리자를 위한 차량 모니터링, 운행 이력, 알림 확인

---

## 2. 기술 스택

- **Framework**: React 18 + Next.js 14 (PWA)
- **UI Library**: Tailwind CSS + shadcn/ui
- **State Management**: Zustand
- **API Client**: Axios
- **Maps**: Kakao Maps SDK
- **Auth**: AWS Cognito (JWT)

---

## 3. AWS 환경 설정

### 3.1 API Gateway

```
Base URL (dev): https://yu9jimmkmc.execute-api.ap-northeast-2.amazonaws.com/prod/v1/mobile
Stage: prod
```

> **참고**: 
> - 개발 환경: `yu9jimmkmc` (trackcar-dev-mobile-api)
> - 프로덕션 환경: `wlvxc40hz9` (trackcar-mobile-api)
> - Lambda Handler가 `/prod/v1/mobile` prefix를 자동으로 처리합니다.

### 3.2 Cognito User Pool

```
Region: ap-northeast-2
User Pool ID: ap-northeast-2_PC62e06zs
Client ID: 3actavsaep6haji08qrga3otov
Cognito Domain: https://trackcar-dev-auth.auth.ap-northeast-2.amazoncognito.com
```

### 3.3 환경 변수

```env
# 개발 환경
NEXT_PUBLIC_API_BASE_URL=https://yu9jimmkmc.execute-api.ap-northeast-2.amazonaws.com/prod/v1/mobile

# 프로덕션 환경
# NEXT_PUBLIC_API_BASE_URL=https://wlvxc40hz9.execute-api.ap-northeast-2.amazonaws.com/prod/v1/mobile

NEXT_PUBLIC_COGNITO_USER_POOL_ID=ap-northeast-2_PC62e06zs
NEXT_PUBLIC_COGNITO_CLIENT_ID=3actavsaep6haji08qrga3otov
NEXT_PUBLIC_COGNITO_REGION=ap-northeast-2
NEXT_PUBLIC_COGNITO_DOMAIN=https://trackcar-dev-auth.auth.ap-northeast-2.amazoncognito.com
```

---

## 4. API Endpoints

> **참고**: API Gateway는 Lambda 수준에서 JWT 인증을 수행합니다.
> 모든 API 호출 시 `Authorization: Bearer {idToken}` 헤더가 필요합니다.

### 4.1 현재 구현된 엔드포인트

| Method | Endpoint | 설명 |
|--------|----------|------|
| GET | `/mobile/vehicles` | 차량 목록 조회 |
| GET | `/mobile/vehicles/{vehicleId}` | 차량 상세 조회 |
| GET | `/mobile/telemetry` | 차량 원격 데이터 조회 |

### 4.2 개발 예정 엔드포인트

| Method | Endpoint | 설명 |
|--------|----------|------|
| GET | `/mobile/dashboard` | KPI, 조치 필요 차량, 최근 알림 |
| GET | `/mobile/trips` | 운행 이력 목록 |
| GET | `/mobile/trips/{tripId}` | 운행 상세 (경로, 이벤트) |
| GET | `/mobile/alerts` | 알림 목록 |
| GET | `/mobile/alerts/{alertId}` | 알림 상세 |
| PATCH | `/mobile/alerts/{alertId}/read` | 알림 확인 |
| GET | `/mobile/groups` | 그룹 목록 |
| GET | `/mobile/me` | 내 정보 조회 |
| PATCH | `/mobile/me/notification-settings` | 알림 설정 변경 |

---

## 5. 공통 응답 형식

### 성공 응답
```json
{
  "success": true,
  "data": { ... },
  "meta": {
    "timestamp": "2026-03-25T10:30:00Z",
    "requestId": "req_abc123"
  }
}
```

### 오류 응답
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "오류 메시지",
    "details": { ... }
  },
  "meta": { ... }
}
```

---

## 6. 인증 (Cognito JWT)

### 6.1 로그인 플로우
1. Cognito User Pool에서 username/password 인증
2. ID Token, Access Token, Refresh Token 수령
3. ID Token을 API Authorization 헤더에 포함

### 6.2 Cognito SDK를 사용한 로그인
```typescript
// lib/auth/cognito.ts
import { CognitoIdentityProviderClient, AdminInitiateAuthCommand } from '@aws-sdk/client-cognito-identity-provider';

const client = new CognitoIdentityProviderClient({ region: 'ap-northeast-2' });

export async function login(username: string, password: string) {
  const command = new AdminInitiateAuthCommand({
    UserPoolId: process.env.NEXT_PUBLIC_COGNITO_USER_POOL_ID,
    ClientId: process.env.NEXT_PUBLIC_COGNITO_CLIENT_ID,
    AuthFlow: 'ADMIN_USER_PASSWORD_AUTH',
    AuthParameters: {
      USERNAME: username,
      PASSWORD: password,
    },
  });

  const response = await client.send(command);
  return {
    idToken: response.AuthenticationResult?.IdToken,
    accessToken: response.AuthenticationResult?.AccessToken,
    refreshToken: response.AuthenticationResult?.RefreshToken,
    expiresIn: response.AuthenticationResult?.ExpiresIn,
  };
}
```

### 6.3 API Client 설정
```typescript
// lib/api/client.ts
import axios from 'axios';

const api = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_BASE_URL,
});

api.interceptors.request.use((config) => {
  const token = localStorage.getItem('idToken');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

api.interceptors.response.use(
  (response) => response,
  async (error) => {
    if (error.response?.status === 401) {
      // 토큰 만료 시 처리
      localStorage.removeItem('idToken');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default api;
```

### 6.4 역할 (Role-based)
| Role | 권한 |
|------|------|
| OWNER | 전체 권한 (Staff 관리, 그룹 관리) |
| STAFF | 차량/운행/알림 조회, 기사 변경 |

### 6.5 테스트 계정
```
Email: admin@test.com
Password: Test1234!

Email: owner@test.com
Password: Test1234!
```

---

## 7. 주요 화면 구성

### 7.1 메뉴 구조 (Bottom Tab Navigation)
```
├── Dashboard (홈)
├── Vehicles (차량)
├── Trips (운행)
├── Alerts (알림)
└── My Account (내 정보)
```

### 7.2 핵심 기능

**Dashboard**
- KPI 요약 (운행 중, 오프라인, 알림)
- 조치 필요 차량 목록
- 최근 알림 목록

**Vehicles**
- 차량 목록 (검색, 필터)
- 차량 상세 (위치, 상태, 기사)
- 실시간 위치 지도

**Trips**
- 운행 이력 목록
- 운행 상세 (경로, 정차, 이벤트)
- 지도에 경로 표시

**Alerts**
- 알림 목록 (심각도별 필터)
- 알림 상세
- 일괄 확인

---

## 8. 주요 UI 컴포넌트

| 컴포넌트 | 용도 |
|----------|------|
| BottomTabBar | 하단 탭 네비게이션 |
| VehicleCard | 차량 목록 카드 |
| TripCard | 운행 이력 카드 |
| AlertItem | 알림 항목 |
| VehicleMap | Kakao Maps 차량 위치 |
| TripRouteMap | Kakao Maps 경로 표시 |
| StatusBadge | 상태 표시 |
| SearchBar | 검색 입력 |
| FilterChips | 필터 칩 |

---

## 9. 상태 관리 (Zustand)

```typescript
// stores/authStore.ts
import { create } from 'zustand';

interface AuthState {
  user: any | null;
  tokens: {
    idToken: string | null;
    accessToken: string | null;
    refreshToken: string | null;
  };
  isAuthenticated: boolean;
  login: (username: string, password: string) => Promise<void>;
  logout: () => void;
  refreshTokens: () => Promise<void>;
}

export const useAuthStore = create<AuthState>((set, get) => ({
  user: null,
  tokens: {
    idToken: null,
    accessToken: null,
    refreshToken: null,
  },
  isAuthenticated: false,
  login: async (username, password) => {
    // Cognito 인증 로직
  },
  logout: () => {
    localStorage.clear();
    set({ user: null, tokens: { idToken: null, accessToken: null, refreshToken: null }, isAuthenticated: false });
  },
  refreshTokens: async () => {
    // Refresh Token으로 토큰 갱신
  },
}));
```

---

## 10. Kakao Maps 연동

### 10.1 Kakao Maps SDK 설치

```bash
npm install react-kakao-maps-sdk
```

### 10.2 환경 변수 설정

```env
NEXT_PUBLIC_KAKAO_MAP_KEY=your_kakao_app_key
```

### 10.3 지도 기본 설정

```typescript
// lib/kakaoMap.ts
import { Map, MapMarker } from 'react-kakao-maps-sdk';

const VehicleMap = ({ lat, lng, vehicles }) => (
  <Map
    center={{ lat, lng }}
    style={{ width: '100%', height: '350px' }}
    level={3}
  >
    {vehicles.map((vehicle) => (
      <MapMarker
        key={vehicle.vehicleId}
        position={{ lat: vehicle.latitude, lng: vehicle.longitude }}
        image={{
          src: '/marker-running.png',
          size: { width: 40, height: 40 },
        }}
        title={vehicle.vehicleNo}
      />
    ))}
  </Map>
);
```

### 10.4 커스텀 마커 (차량 상태별)

```typescript
// 차량 상태별 마커 아이콘 (public/markers/ 폴더에 저장)
const markerImages = {
  RUNNING: { src: '/markers/marker-running.png', size: { width: 40, height: 40 } },
  IDLE: { src: '/markers/marker-idle.png', size: { width: 40, height: 40 } },
  OFFLINE: { src: '/markers/marker-offline.png', size: { width: 40, height: 40 } },
  START: { src: '/markers/marker-start.png', size: { width: 32, height: 44 } },
  END: { src: '/markers/marker-end.png', size: { width: 32, height: 44 } },
};

const VehicleMarker = ({ vehicle }) => {
  const image = markerImages[vehicle.tripStatus] || markerImages.OFFLINE;
  
  return (
    <MapMarker
      position={{ lat: vehicle.latitude, lng: vehicle.longitude }}
      image={{ src: image.src, size: image.size }}
      title={`${vehicle.vehicleNo} - ${vehicle.tripStatus}`}
    />
  );
};
```

> **마커 이미지**: `public/markers/` 폴더에 PNG 파일 저장
> - `marker-running.png` (운행 중 - 파란색)
> - `marker-idle.png` (정차 중 - 노란색)
> - `marker-offline.png` (오프라인 - 회색)
> - `marker-start.png` (출발점 - 초록색)
> - `marker-end.png` (도착점 - 빨간색)

---

## 11. API 호출 예시

### 11.1 차량 목록 조회
```typescript
// hooks/useVehicles.ts
import { useState, useEffect } from 'react';
import api from '@/lib/api/client';

interface Vehicle {
  vehicleId: string;
  status: string;
  latitude?: number;
  longitude?: number;
}

export function useVehicles() {
  const [vehicles, setVehicles] = useState<Vehicle[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    async function fetchVehicles() {
      try {
        const response = await api.get('/mobile/vehicles');
        setVehicles(response.data.data.vehicles || []);
      } catch (err) {
        setError(err.message);
      } finally {
        setLoading(false);
      }
    }
    fetchVehicles();
  }, []);

  return { vehicles, loading, error };
}
```

### 11.2 차량 원격 데이터 조회
```typescript
// hooks/useTelemetry.ts
import { useState, useEffect } from 'react';
import api from '@/lib/api/client';

interface Telemetry {
  vehicleId: string;
  latitude: number;
  longitude: number;
  speed: number;
  lastUpdated: string;
}

export function useTelemetry(vehicleId?: string) {
  const [telemetry, setTelemetry] = useState<Telemetry | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    if (!vehicleId) return;

    async function fetchTelemetry() {
      try {
        const response = await api.get(`/mobile/telemetry/${vehicleId}`);
        setTelemetry(response.data.data);
      } catch (err) {
        console.error('Failed to fetch telemetry:', err);
      } finally {
        setLoading(false);
      }
    }
    fetchTelemetry();

    // 실시간 업데이트를 위한 폴링 (10초마다)
    const interval = setInterval(fetchTelemetry, 10000);
    return () => clearInterval(interval);
  }, [vehicleId]);

  return { telemetry, loading };
}
```

### 10.5 경로 그리기 (Polyline)

```typescript
import { Map, MapMarker, Polyline } from 'react-kakao-maps-sdk';

const TripRouteMap = ({ routePoints }) => (
  <Map
    center={routePoints[0]}
    style={{ width: '100%', height: '400px' }}
  >
    <Polyline
      path={routePoints}
      strokeWeight={5}
      strokeColor="#0068FF"
      strokeOpacity={0.8}
      strokeStyle="solid"
    />
    {routePoints.map((point, index) => (
      <MapMarker
        key={index}
        position={point}
        icon={{
          src: index === 0 ? '/marker-start.png' : 
               index === routePoints.length - 1 ? '/marker-end.png' : '/marker-point.png',
          size: { width: 24, height: 35 },
        }}
      />
    ))}
  </Map>
);
```

### 10.6 클러스터링 (다수 마커)

```typescript
import { MapMarker } from 'react-kakao-maps-sdk';

// 간단한 클러스터링 구현
const MarkerCluster = ({ vehicles }) => {
  const [zoomLevel, setZoomLevel] = useState(3);
  
  // 줌 레벨에 따라 마커 표시/숨김
  const visibleVehicles = zoomLevel >= 5 ? vehicles : vehicles.slice(0, 10);
  
  return (
    <>
      {visibleVehicles.map((vehicle) => (
        <MapMarker
          key={vehicle.vehicleId}
          position={{ lat: vehicle.latitude, lng: vehicle.longitude }}
        />
      ))}
      {zoomLevel < 5 && (
        <div className="cluster-badge">+{vehicles.length - 10}</div>
      )}
    </>
  );
};
```

---

## 11. 개발 시작

### 11.1 PWA (React)
```bash
npx create-next-app@latest trackcar-mobile
cd trackcar-mobile
npm install axios zustand react-kakao-maps-sdk date-fns clsx tailwind-merge lucide-react
```

### 11.2 폴더 구조
```
trackcar-mobile/
├── app/
│   ├── layout.tsx
│   ├── page.tsx              # Dashboard
│   ├── vehicles/
│   ├── trips/
│   ├── alerts/
│   └── my-account/
├── components/
│   ├── ui/
│   ├── layout/
│   ├── map/                  # Kakao Maps 컴포넌트
│   │   ├── VehicleMap.tsx
│   │   ├── TripRouteMap.tsx
│   │   └── MarkerCluster.tsx
│   └── features/
├── lib/
│   ├── api/
│   └── utils/
├── stores/
├── types/
└── public/
    └── manifest.json
```

### 11.3 Kakao Maps SDK 로드

```typescript
// app/layout.tsx 또는 lib/kakaoMap.ts
import { useKakaoLoader } from 'react-kakao-maps-sdk';

export default function RootLayout({ children }) {
  useKakaoLoader({
    appkey: process.env.NEXT_PUBLIC_KAKAO_MAP_KEY || '',
  });

  return (
    <html lang="ko">
      <body>{children}</body>
    </html>
  );
}
```

### 11.4 PWA 설정
```json
// public/manifest.json
{
  "name": "TrackCar",
  "short_name": "TrackCar",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#000000",
  "icons": [...]
}
```

---

## 12. 참고 문서

- TrackCar Mobile App API Specification
- TrackCar Mobile App 화면 상세 설계
- TrackCar Mobile App 요구사항 및 기능 명세서

---

## 13. 질문/문의

실시간으로 질문사항이 있으면 TrackCar 개발팀에 문의하세요.
