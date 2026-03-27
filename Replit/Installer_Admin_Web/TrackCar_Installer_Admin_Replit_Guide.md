# TrackCar Installer Admin Web - Replit 개발 가이드

## 1. 프로젝트 개요

**프로젝트명**: TrackCar Installer Admin Web
**플랫폼**: Installer Admin (설치 관리자용 웹 대시보드)
**목적**: DTG 단말기 설치, 차량/기사/오너 관리, 검증, 앱 활성화 관리

---

## 2. 기술 스택

- **Framework**: React 18 + Next.js 14 (App Router)
- **UI Library**: Tailwind CSS + shadcn/ui
- **State Management**: Zustand
- **API Client**: Axios
- **Charts**: Recharts
- **Auth**: AWS Cognito (JWT)

---

## 3. AWS 환경 설정

### 3.1 API Gateway

```
Base URL: https://a1xpskzj48.execute-api.ap-northeast-2.amazonaws.com/prod/v1/api/admin
Stage: prod
```

> **참고**: Lambda Handler가 `/v1/api/admin/` prefix를 자동으로 처리합니다.
> API Gateway 경로가 `/prod/v1/api/admin`으로 설정되어 있습니다.

### 3.2 Cognito User Pool

```
Region: ap-northeast-2
User Pool ID: ap-northeast-2_PC62e06zs
Client ID: 3actavsaep6haji08qrga3otov
Cognito Domain: https://trackcar-dev-auth.auth.ap-northeast-2.amazoncognito.com
```

### 3.3 환경 변수 (.env.local)

```env
NEXT_PUBLIC_API_BASE_URL=https://a1xpskzj48.execute-api.ap-northeast-2.amazonaws.com/prod/v1/api/admin
NEXT_PUBLIC_COGNITO_USER_POOL_ID=ap-northeast-2_PC62e06zs
NEXT_PUBLIC_COGNITO_CLIENT_ID=3actavsaep6haji08qrga3otov
NEXT_PUBLIC_COGNITO_REGION=ap-northeast-2
NEXT_PUBLIC_COGNITO_DOMAIN=https://trackcar-dev-auth.auth.ap-northeast-2.amazoncognito.com
```

---

## 4. API Endpoints

### 4.1 Dashboard
| Method | Endpoint | 설명 |
|--------|----------|------|
| GET | `/dashboard` | KPI, 차트, 이상 차량 목록 |

### 4.2 Owner
| Method | Endpoint | 설명 |
|--------|----------|------|
| GET | `/owners` | Owner 목록 조회 |
| GET | `/owners/{ownerId}` | Owner 상세 |
| POST | `/owners` | Owner 생성 |
| PUT | `/owners/{ownerId}` | Owner 수정 |
| POST | `/owners/check-duplicate` | 중복 확인 |

### 4.3 Group
| Method | Endpoint | 설명 |
|--------|----------|------|
| GET | `/groups` | 그룹 목록 |
| POST | `/groups` | 그룹 생성 |
| PUT | `/groups/{groupId}` | 그룹 수정 |

### 4.4 Device (DTG)
| Method | Endpoint | 설명 |
|--------|----------|------|
| GET | `/devices` | DTG 목록 |
| GET | `/devices/{deviceId}` | DTG 상세 |
| POST | `/devices` | DTG 등록 |
| PUT | `/devices/{deviceId}` | DTG 수정 |

### 4.5 Vehicle
| Method | Endpoint | 설명 |
|--------|----------|------|
| GET | `/vehicles` | 차량 목록 |
| GET | `/vehicles/{vehicleId}` | 차량 상세 |
| POST | `/vehicles` | 차량 등록 |
| PUT | `/vehicles/{vehicleId}` | 차량 수정 |

### 4.6 Driver
| Method | Endpoint | 설명 |
|--------|----------|------|
| GET | `/drivers` | 기사 목록 |
| GET | `/drivers/{driverId}` | 기사 상세 |
| POST | `/drivers` | 기사 생성 |
| PUT | `/drivers/{driverId}` | 기사 수정 |

### 4.7 Mapping
| Method | Endpoint | 설명 |
|--------|----------|------|
| GET | `/mappings/candidates` | 매핑 후보 조회 |
| POST | `/mappings/bundle` | 묶음 매핑 저장 |

### 4.8 Verification
| Method | Endpoint | 설명 |
|--------|----------|------|
| GET | `/verifications` | 검증 목록 |
| GET | `/verifications/{verificationId}` | 검증 상세 |
| POST | `/verifications/run` | 검증 실행 |

### 4.9 App Activation
| Method | Endpoint | 설명 |
|--------|----------|------|
| GET | `/app-activations` | 앱 활성화 목록 |
| GET | `/app-activations/{activationId}` | 상세 |
| POST | `/app-activations/{activationId}/invite` | 초대 발송 |
| POST | `/app-activations/{activationId}/resend-invite` | 재발송 |

### 4.10 Account Linking
| Method | Endpoint | 설명 |
|--------|----------|------|
| GET | `/accounts/candidates` | 계정 후보 검색 |
| POST | `/drivers/{driverId}/link-account` | 기존 계정 연결 |
| POST | `/drivers/{driverId}/create-linked-account` | 새 계정 생성 및 연결 |

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

### 공통 오류 코드
| HTTP Status | Error Code |
|-------------|------------|
| 400 | VALIDATION_ERROR |
| 401 | UNAUTHORIZED |
| 403 | FORBIDDEN |
| 404 | NOT_FOUND |
| 409 | CONFLICT |
| 500 | INTERNAL_ERROR |

---

## 6. 인증 (Cognito JWT)

### 6.1 로그인 플로우
1. Cognito User Pool에서 인증
2. JWT ID Token 수령
3. API 호출 시 `Authorization: Bearer {token}` 헤더 포함

### 6.2 사용 예시
```typescript
// api/client.ts
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
```

### 6.3 역할 (Role-based)
| Role | 권한 |
|------|------|
| Installer | 설치 관련 CRUD |
| Operator | 설치 + Owner/Group 관리 |
| Admin | 전체 권한 |

---

## 7. 주요 화면 구성

### 7.1 메뉴 구조
```
├── Dashboard
├── Owner Management
│   ├── Owner List
│   └── Owner Detail
├── Device (DTG) Management
│   ├── Device List
│   └── Device Detail
├── Vehicle Management
│   ├── Vehicle List
│   └── Vehicle Detail
├── Driver Management
│   ├── Driver List
│   └── Driver Detail
├── Mapping
│   └── Quick Mapping
├── Verification
│   ├── Verification List
│   └── Verification Detail
├── App Activation
│   └── Activation Management
└── Settings
```

### 7.2 핵심 기능

**Dashboard**
- 설치 현황 KPI (총 설치, 온라인, 미연결)
- 설치 추이 차트
- 디바이스 상태 분포
- 이상 차량 목록

**Quick Mapping**
- DTG-차량-기사 묶음 매핑
- 매핑 후보 자동 필터링

**Verification**
- GPS 수신 확인
- 하트비트 확인
- 앱 로그인 확인

---

## 8. 주요 UI 컴포넌트

| 컴포넌트 | 용도 |
|----------|------|
| DataTable | 목록 조회 (페이지네이션, 정렬, 필터) |
| StatusBadge | 상태 표시 (ACTIVE, INACTIVE 등) |
| StatCard | KPI 카드 |
| Chart (Recharts) | 차트 (Line, Bar, Pie) |
| SearchInput | 검색 필터 |
| Modal | 모달 대화상자 |
| ConfirmDialog | 확인/취소 대화상자 |

---

## 9. 상태 관리 (Zustand)

```typescript
// stores/authStore.ts
import { create } from 'zustand';

interface AuthState {
  user: any | null;
  isAuthenticated: boolean;
  login: (token: string) => void;
  logout: () => void;
}

export const useAuthStore = create<AuthState>((set) => ({
  user: null,
  isAuthenticated: false,
  login: (token) => set({ isAuthenticated: true }),
  logout: () => set({ user: null, isAuthenticated: false }),
}));
```

---

## 10. 개발 시작

### 10.1 Replit에서 프로젝트 생성
```bash
npx create-next-app@latest trackcar-admin
cd trackcar-admin
npm install axios zustand recharts date-fns
```

### 10.2 폴더 구조
```
trackcar-admin/
├── app/
│   ├── layout.tsx
│   ├── page.tsx
│   ├── dashboard/
│   ├── owners/
│   ├── devices/
│   ├── vehicles/
│   ├── drivers/
│   ├── mappings/
│   ├── verifications/
│   └── app-activations/
├── components/
│   ├── ui/           # shadcn/ui
│   ├── layout/
│   └── features/
├── lib/
│   ├── api/
│   └── utils/
├── stores/
└── types/
```

### 10.3 API Client 설정
```typescript
// lib/api/client.ts
export const apiClient = createApiClient();
```

---

## 11. 참고 문서

- TrackCar Installer Admin Web API Specification
- TrackCar Installer Admin 화면 상세 설계
- TrackCar Installer Admin 요구사항 및 기능 명세서

---

## 12. 질문/문의

실시간으로 질문사항이 있으면 TrackCar 개발팀에 문의하세요.
