<!-- FILE: README.md -->

# Kyobo Electronics Inventory & AI Web

## 프로젝트 개요

교보문고 오프라인 지점에서 판매 중인 **전자기기(전자책 리더기, 태블릿, 주변기기)** 의 재고 및 매대 위치를 실시간으로 제공하고, AI 기반 수요 예측·추천 기능을 통합한 웹 애플리케이션입니다.

### 핵심 기능

1. **실시간 재고 현황** – REST API + WebSocket으로 수량 변동을 즉시 반영
2. **매장 내 위치 시각화** – SVG/Canvas 기반 매장 지도 위에 상품 배치 오버레이
3. **AI 수요 예측** – Prophet 기반 일/주간 재고 소진 예측, 안전재고 알림
4. **자연어 Q\&A** – GPT‑4o를 통해 “강남점에 오늘 살 수 있어?” 같은 질의 처리
5. **개인화 추천** – 사용자의 독서 패턴, 예산, 디바이스 사용 목적을 바탕으로 기기 추천
6. **백오피스 대시보드** – 관리자용 재고·판매 분석, CSV 업로드, 알림 템플릿 관리

### 기술 스택

| Layer           | Tech                                                | 비고             |
| --------------- | --------------------------------------------------- | -------------- |
| **Frontend**    | Next.js 15, TypeScript, TailwindCSS, SWR            | PWA 지원         |
| **Backend**     | FastAPI, Python 3.12                                | ASGI, uvicorn  |
| **DB**          | PostgreSQL 16 + PostGIS                             | 위치 좌표 저장       |
| **Cache/Queue** | Redis, Celery                                       | 비동기 예측 작업      |
| **AI**          | OpenAI GPT‑4o, Prophet, scikit‑learn                |                |
| **Infra**       | Docker, GitHub Actions, AWS ECS Fargate, CloudFront | IaC: Terraform |

### 폴더 구조(간략)

```
.
├── app/                # FastAPI src
├── web/                # Next.js src
├── docs/               # Markdown 문서
├── data/               # 샘플 CSV/SQL 덤프
└── infra/              # Terraform, Docker, GitHub Actions
```

### 시작하기(로컬 개발)

```bash
git clone https://github.com/<YOUR-ORG>/kyobo-electronics.git
cd kyobo-electronics
cp .env.example .env   # 환경 변수 수정
docker compose up -d   # API + DB + Redis
npm install --prefix web
npm run dev --prefix web
```

---

<!-- FILE: DATA_SCHEMA.md -->

# 데이터 스키마

## ERD

* **stores** (store\_id PK, name, address, tel, lat, lon, opening\_hours)
* **devices** (device\_id PK, sku, name, category, brand, price, specs\_json)
* **inventory** (inventory\_id PK, store\_id FK, device\_id FK, quantity, last\_updated)
* **locations** (location\_id PK, store\_id FK, device\_id FK, floor, zone, shelf, x\_px, y\_px)
* **sales** (sale\_id PK, store\_id FK, device\_id FK, qty, sold\_at)
* **predictions** (pred\_id PK, store\_id FK, device\_id FK, date, expected\_demand, created\_at)

### 핵심 필드 설명

| 테이블         | 필드               | 타입           | 설명           |
| ----------- | ---------------- | ------------ | ------------ |
| stores      | lat/lon          | decimal(9,6) | WGS‑84 좌표    |
| locations   | x\_px, y\_px     | int          | SVG 좌표계 픽셀 값 |
| inventory   | quantity         | smallint     | 현재 수량        |
| predictions | expected\_demand | numeric      | 하루 기준 예상 판매량 |

### 샘플 쿼리

```sql
-- 강남점(KNG001)에서 3일 내 품절 위험이 있는 기기
SELECT d.sku, d.name, i.quantity, p.expected_demand
FROM inventory i
JOIN devices d ON d.device_id = i.device_id
JOIN predictions p ON p.device_id = i.device_id
WHERE i.store_id = 'KNG001'
  AND p.date = CURRENT_DATE
  AND i.quantity < p.expected_demand * 3;
```

---

<!-- FILE: LOCATION_MAPPING.md -->

# 매장 위치 매핑 가이드

## 매장 지도 파일

* 각 지점별 **SVG** 파일(`stores/<store_id>.svg`)을 사용
* `<g id="device-{device_id}">` 형태의 그룹을 생성하여 디바이스 위치를 매핑
* 좌표는 `locations` 테이블의 `x_px`, `y_px`와 일치

## 프론트엔드 렌더링

```tsx
import { ReactComponent as Map } from "@/maps/KNG001.svg";

<Map className="w-full h-auto">
  {inventory.map(item => (
    <circle key={item.device_id}
            cx={item.x_px}
            cy={item.y_px}
            r={12}
            className={item.quantity > 0 ? "fill-green-500" : "fill-red-600"}
            data-tip={`${item.name} 재고: ${item.quantity}대`} />
  ))}
</Map>
```

---

<!-- FILE: AI_MODULE.md -->

# AI 모듈 설계

## 수요 예측 파이프라인

1. **데이터 수집** – `sales`, `inventory` 24개월 히스토리
2. **전처리** – 결측치 보간, 프로모션·연휴 Feature 생성
3. **모델 학습**

   * 베이스라인: Prophet (daily)
   * 향상 모델: XGBoost + 시계열 Cross‑Validation
4. **예측 결과 저장** – `predictions` 테이블
5. **알림 트리거** – `quantity < expected_demand * safety_days` 조건 시 Slack & Email

## 자연어 Q\&A

* **Embedding**: `text-embedding-3-small`로 상품·매장 정보를 벡터화
* **검색**: pgvector `INNER_PRODUCT` KNN, Top‑k=8
* **LLM**: GPT‑4o, System Prompt 예시

  > “당신은 교보문고 매장 직원입니다. 사용자 질문에 친절하고 간결하게 답하세요.”

## 추천 알고리즘

* 협업필터링(NCF) + 콘텐츠 기반 가중치
* 사용자의 **독서 장르·페이지 수·예산** 벡터 + 디바이스 스펙(해상도, 무게, 가격) 코사인 유사도

---

<!-- FILE: API_SPEC.md -->

# REST API 명세

| 메서드    | 엔드포인트                             | 설명              |
| ------ | --------------------------------- | --------------- |
| `GET`  | `/stores`                         | 지점 목록           |
| `GET`  | `/stores/{id}/inventory`          | 지점별 재고          |
| `GET`  | `/devices/{id}`                   | 기기 상세           |
| `GET`  | `/search`                         | `q` 파라미터 자연어 검색 |
| `POST` | `/predict/run`                    | 수요 예측 배치 트리거    |
| `GET`  | `/predict/{store_id}/{device_id}` | 예측 결과           |
| `WS`   | `/ws/updates`                     | 재고 변동 스트림       |

### 예시 응답

```jsonc
{
  "store_id": "KNG001",
  "device_id": "E‑INK‑AURA",
  "quantity": 12,
  "location": { "floor": 1, "zone": "A3", "shelf": "E‑Readers" },
  "updated_at": "2025‑07‑07T10:15:22+09:00"
}
```

---

<!-- FILE: DEPLOYMENT.md -->

# 배포 & 운영

## CI/CD

1. **GitHub Actions**

   * `push main` → 테스트 → Docker 이미지 빌드 → ECR 푸시
2. **Terraform**

   * VPC, RDS(PostgreSQL), ElastiCache(Redis), ECS Fargate, ALB
3. **CDN**

   * CloudFront + S3 정적 자산 캐시

## 모니터링

* **Prometheus + Grafana**: API Latency, Error Rate
* **AWS CloudWatch**: ECS, RDS 지표
* **Sentry**: 프론트엔드 오류 추적

## 백업

* RDS 자동 스냅샷(7일), S3 로티션

---

<!-- FILE: CONTRIBUTING.md -->

# Contributing Guide

## 브랜치 전략

* `main`: 배포용
* `dev`: 통합 개발
* `feat/*`: 기능
* `fix/*`: 패치

## 커밋 메시지 컨벤션

`type(scope): subject`

* `feat`, `fix`, `docs`, `refactor`, `test`, `chore`

## PR 규칙

* 최소 1명 리뷰어 승인
* CI 통과 필수
* 유닛 테스트 추가
