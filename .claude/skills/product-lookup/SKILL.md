---
name: product-lookup
description: 스킨케어 제품명(쉼표 구분)을 입력받아 WebSearch로 직접 정보를 수집한 뒤 products.json 스키마로 출력하고 파일에 추가한다. 트리거: "제품 찾아줘", "제품 조회", "/product-lookup", 제품명 나열 후 "json으로" 요청.
---

## 목적

제품명 하나 이상을 받아 WebSearch로 올리브영 및 화해·글로우픽 정보를 수집하고, products.json 스키마에 맞는 JSON 배열을 생성한 뒤 파일에 추가한다. Python 스크립트나 외부 API 없이 Claude가 직접 동작한다.

---

## 절차

### 1단계 — 입력 파싱

사용자 메시지에서 쉼표(`,`)로 구분된 제품명 목록을 추출한다. 앞뒤 공백과 빈 항목은 제거한다.

```
조회할 제품: N개
1. [제품명1]
2. [제품명2]
```

### 2단계 — 제품별 WebSearch 병렬 수집

각 제품에 대해 아래 두 쿼리를 병렬로 검색한다.

1. `{제품명} 올리브영 가격 성분`
2. 성분 정보가 부족하면 `{제품명} 전성분 화해`

**추출 대상 필드**

| 필드 | 추출 방법 |
|------|-----------|
| `brand` | 공식 브랜드명 (한국어 표기) |
| `name` | 올리브영 공식 제품명 (용량 포함) |
| `cat` | 아래 카테고리 분류표 참고 |
| `price` | 올리브영 할인가 (정수, 원 단위, 없으면 null) |
| `originalPrice` | 정가 (할인 없으면 null) |
| `priceApprox` | 가격이 불확실하면 true |
| `ingredients` | 핵심 성분 배열 — amount는 퍼센트 명시 시만 기재 |
| `badges` | 아래 배지 키 목록에서 선택 |
| `warning` | 사용 주의사항 (없으면 null) |
| `usage` | 의약품 사용법 (rx가 아니면 null) |
| `isDaiso` | 다이소 제품이면 true |
| `isRx` | 약국 의약품이면 true |
| `brandColor` | rx만 설정, 나머지 null |
| `image` | `브랜드영문_핵심단어.jpg` 소문자 스네이크케이스 |
| `query` | 올리브영 검색어 (용량·기획 제외) |

---

### 카테고리 분류표

| cat 값 | 해당 제품 유형 |
|--------|---------------|
| `owned` | 현재 보유 중인 제품 (사용자가 명시할 때만) |
| `cleansingfoam` | 클렌징폼 · 클렌징젤 · 클렌징바 |
| `milk` | 클렌징밀크 · 클렌징오일 · 클렌징크림 |
| `cica` | 시카/센텔라 중심 토너 · 세럼 · 크림 |
| `houttuynia` | 어성초 중심 세럼 · 토너 |
| `panthenol` | 판테놀/비타민B5 또는 히알루론산 중심 앰플 · 세럼 · 크림 |
| `bha` | BHA(살리실산) 중심 패드 · 토너 · 세럼 |
| `sunscreen` | 선크림 · 선세럼 · 선스틱 |
| `rx` | 약국 일반의약품 |

---

### 배지 키 목록

- **amber**: `추천`, `예산우선`
- **purple (★ 접두)**: `홍조픽`, `가성비픽`, `입문픽`, `장벽픽`, `6월한정픽`, `상시픽`, `이중효과픽`, `최저가픽`
- **green**: `장벽강화`, `무기자차`
- **pink**: `보유`
- **blue**: `다이소`
- **red**: `의약품`

---

### 3단계 — id 부여

`products.json` 파일을 읽어 마지막 id를 확인하고 +1부터 순서대로 부여한다.

### 4단계 — JSON 출력

```json
[
  {
    "id": 34,
    "brand": "브랜드명",
    "name": "정식 제품명 용량",
    "image": "brand_keyword.jpg",
    "cat": "카테고리",
    "price": 15000,
    "originalPrice": null,
    "priceApprox": false,
    "ingredients": [
      {"name": "성분명", "amount": "10%"},
      {"name": "성분명2", "amount": null}
    ],
    "badges": ["추천"],
    "warning": null,
    "usage": null,
    "isDaiso": false,
    "isRx": false,
    "brandColor": null,
    "query": "브랜드 핵심제품명"
  }
]
```

### 5단계 — 미확인 필드 표시

```
⚠ 미확인 필드
- [제품명]: ingredients (전성분 미확인), price (가격 미확인)
```

### 6단계 — products.json 추가

사용자가 "추가해줘" / "그렇게 해" 등으로 확인하면 `products.json`을 읽고 배열 끝에 append 후 저장한다.
