# The Cat API 사용법 정리

> 출처: https://thecatapi.com/ , https://docs.thecatapi.com/ , https://api.thecatapi.com/openapi-json
> 정리일: 2026-11-09
> ⚠️ 표시 = 공식 문서끼리 내용이 다르거나 문서에서 확인하지 못한 항목
> ✅ 표시 = 2026-11-09 Free 요금제 키로 직접 호출해서 확인한 항목 (결과 요약은 7장)

---

## 1. 기본 정보

| 항목 | 내용 |
|---|---|
| Base URL | `https://api.thecatapi.com/v1` |
| 인증 방식 | 요청 헤더 `x-api-key: YOUR-API-KEY` |
| API 키 발급 | https://thecatapi.com/signup 에서 가입하면 이메일로 키가 옵니다 |
| 응답 형식 | JSON |
| 이미지 CDN | `https://cdn2.thecatapi.com/images/{id}.jpg` |

⚠️ OpenAPI 스펙에는 서버 주소가 `https://beta-api.thecatapi.com/v1`로 적혀 있습니다. 가이드 문서는 모두 `api.thecatapi.com`을 쓰니 이 주소를 쓰면 됩니다.
⚠️ `?api_key=` 쿼리 파라미터 방식은 문서에 나오지 않습니다. 헤더 방식만 쓰세요.

### 키 없이 쓸 수 있는 엔드포인트 (OpenAPI 스펙 기준)
- `GET /images/search`
- `GET /images/{id}`
- `GET /votes`, `GET /votes/{vote_id}`

그 밖의 엔드포인트(breeds, categories, favourites, facts, 업로드 등)는 모두 `x-api-key`가 필요합니다.

---

## 2. 요금제와 요청 한도

| 요금제 | 가격 | 월 요청 수 | 비고 |
|---|---|---|---|
| Free | $0 | 10,000회 | 분당 10회까지. 품종은 핵심 필드만 제공 |
| Startup | $99/월 | 100,000회 | 품종 상세 점수·특성, 고양이 상식(facts), 다국어(`lang`) 제공 |
| Enterprise | 별도 문의 | 무제한 | 건강 팁, 사진 분석 등 제공 |

- 내 키에서 켜진 기능 확인: `GET /accounts/features`
- 업로드 한도 확인: `GET /accounts/quota` → `{"quota": 25, "usage": 12, "remaining": 13}`
- 파일 크기 제한: 사진 50MB, 동영상 200MB

---

## 3. 에러 응답

```json
{
  "statusCode": 404,
  "timestamp": "...",
  "path": "/v1/...",
  "message": "...",
  "error": "Not Found"
}
```

| 코드 | 의미 |
|---|---|
| 400 | 잘못된 요청 또는 한도 초과 |
| 401 | 비활성 계정 |
| 403 | 키가 없거나 잘못됨, 또는 내 요금제에 없는 기능 |
| 404 | 리소스 없음 |
| 429 | 요청 한도 초과 |
| 500 / 503 | 서버 오류 |

**재시도 권장 방식:** 429·500·503에서만 재시도합니다. 대기 시간은 1초 → 2초 → 4초 → 8초로 늘리고(지터 추가), 4~5번 실패하면 멈춥니다.

---

## 4. 엔드포인트

### 4-1. 이미지

#### `GET /images/search` — 고양이 이미지 검색 (키 없이 사용 가능)

| 파라미터 | 타입 | 설명 |
|---|---|---|
| `limit` | number | 1~100. 기본값 1 (⚠️ 설명에는 10이라고도 적혀 있음) |
| `page` | number | 0부터 시작. 기본값 0 |
| `order` | string | `ASC` / `DESC` / `RANDOM` (⚠️ 가이드 표에는 `RAND`로 적혀 있음) |
| `has_breeds` | boolean | 품종 정보가 있는 이미지만 |
| `breed_ids` | string | 품종 ID를 쉼표로 구분 (예: `beng,abys`) |
| `category_ids` | string | ⚠️ 카테고리 필터. 스펙 본문에서는 확인하지 못함. ✅ `GET /categories`가 빈 배열을 돌려주므로 지금은 쓸 데가 없음 |
| `mime_types` | string | `jpg,png,gif` |
| `size` | string | `thumb`/`small`/`med`/`full` (호환용이라 실제로는 무시됨) |
| `sub_id` | string | 사용자 구분용 ID |

```js
const res = await fetch(
  "https://api.thecatapi.com/v1/images/search?limit=10&has_breeds=1",
  { headers: { "x-api-key": API_KEY } }
);
const cats = await res.json();
```

응답 예시:
```json
[
  {
    "id": "KWdLHmOqc",
    "url": "https://cdn2.thecatapi.com/images/KWdLHmOqc.jpg",
    "width": 3114,
    "height": 2609,
    "breeds": []
  }
]
```

#### `GET /images/{id}` — 이미지 하나 조회 (키 없이 사용 가능)
```
GET https://api.thecatapi.com/v1/images/KWdLHmOqc
```

#### `GET /images` — 내가 업로드한 이미지 목록
- `?sub_id=xxx`로 걸러볼 수 있습니다.
- ⚠️ 가이드에만 나오고 스펙에는 없습니다.

#### `POST /images/upload` — 이미지 업로드 (비동기 처리)
`multipart/form-data`로 보냅니다.

| 필드 | 설명 |
|---|---|
| `file` | 이미지 파일 |
| `type` | `FILE` 또는 `base64` (스펙상 필수) |
| `image` | base64 문자열 (`type=base64`일 때) |
| `sub_id` | 사용자 구분용 ID (선택) |

```js
const form = new FormData();
form.append("file", fileInput.files[0]);
form.append("type", "FILE");
form.append("sub_id", "my-user-123");

await fetch("https://api.thecatapi.com/v1/images/upload", {
  method: "POST",
  headers: { "x-api-key": API_KEY }, // Content-Type은 브라우저가 알아서 지정하므로 넣지 않음
  body: form,
});
```

- `POST /images/upload-sync`: 분석이 끝날 때까지 기다렸다가 응답합니다. `auto_crop` 옵션이 있고 기본값은 true입니다.
- `GET /images/{id}/status`: 처리 상태를 확인합니다. 상태값은 `to_be_reviewed` → `clean` → `labelled`이며, 거절되면 `unsafe` / `does_not_contain_species`, 수동 검토 대상이면 `flagged`가 됩니다.
- `GET /images/{id}/labels`: 품종 판별 결과를 돌려줍니다.
- `DELETE /images/{id}`: 업로드한 이미지를 삭제합니다.

---

### 4-2. 품종 (Breeds) — 키 필요

#### `GET /breeds` — 품종 전체 목록
| 파라미터 | 설명 |
|---|---|
| `limit` | 1~1000. 기본값 1000 |
| `page` | 기본값 0 |
| `order` | `ASC`/`DESC`/`RANDOM` |
| `breed_groups` | 예: `Short-haired,Rex` |
| `lang` | 언어 코드 (Startup 이상) |

#### `GET /breeds/search?q=siamese` — 품종 이름 검색
- `q`: 1~30자. 이름 일부나 별칭으로도 검색됩니다.

#### `GET /breeds/{id}` — 품종 상세
- 품종 ID는 `abys`, `beng`, `siam` 같은 짧은 코드입니다.

응답 예시 (일부만):
```json
{
  "id": "abys",
  "name": "Abyssinian",
  "temperament": "Active, Energetic, Independent, Intelligent, Gentle",
  "origin": "Egypt",
  "country_code": "EG",
  "description": "Medium-sized, elegant cat ...",
  "life_span": "14-17",
  "reference_image_id": "KWdLHmOqc",
  "image": { "id": "KWdLHmOqc", "url": "https://cdn2.thecatapi.com/images/KWdLHmOqc.jpg" },
  "adaptability": 3,
  "affection_level": 3,
  "energy_level": 3,
  "intelligence": 4
}
```
> 1~5점 점수 필드와 boolean 특성 필드는 Startup 이상에서만 옵니다. Free는 핵심 필드만 받습니다.

✅ **Free 요금제에서 실제로 오는 필드:**
`id`, `name`, `species_id`, `life_span`, `temperament`, `origin`, `country_codes`, `country_code`, `description`, `bred_for`, `perfect_for`, `breed_group`, `history`, `alt_names`, `reference_image_id`, `weight`(`{imperial, metric}`), `height`(`{imperial, metric}`), `image`(`{id, url, width, height}`)
- `adaptability`, `energy_level` 같은 점수 필드는 **오지 않습니다**.
- `weight`, `height`는 문서 예시에는 없지만 실제 응답에 들어 있습니다.
- 대표 이미지가 없는 품종도 있습니다(예: `thai`). 이때는 `reference_image_id`가 비어 있고 `image` 필드가 없으므로, 화면에 대체 이미지를 준비하세요.
- `breeds/search?q=siam`은 이름과 별칭을 함께 검색해 Siamese, Balinese, Thai를 돌려줍니다.

#### 그 밖의 품종 엔드포인트
- `GET /breeds/breed-groups`: 품종 그룹 목록 (Short-haired, Long-haired, Rex, Hairless 등)
- `GET /breeds/{id}/colours`: 품종별 색상과 무늬

> 💡 특정 품종 사진을 가져오려면 `GET /images/search?breed_ids=abys&limit=10`을 쓰세요.

---

### 4-3. 카테고리 — 키 필요
- `GET /categories`: `limit`(기본값 10), `page`, `order`
- `GET /categories/{id}`

✅ 실제 호출 결과, `GET /categories`는 **빈 배열 `[]`**을 돌려줍니다. 카테고리로 이미지를 거르는 기능 대신 품종(`breed_ids`)이나 품종 그룹으로 거르세요.

---

### 4-4. 투표 (Votes)

| 메서드 | 경로 | 키 | 설명 |
|---|---|---|---|
| GET | `/votes` | 불필요 | 투표 목록 (`sub_id`, `limit`, `page`, `order`) |
| GET | `/votes/{vote_id}` | 불필요 | 투표 하나 조회 |
| POST | `/votes` | 필요 | 투표하기 |

```js
await fetch("https://api.thecatapi.com/v1/votes", {
  method: "POST",
  headers: { "Content-Type": "application/json", "x-api-key": API_KEY },
  body: JSON.stringify({ image_id: "KWdLHmOqc", value: 1, sub_id: "my-user-123" }),
});
```
- 투표는 삭제할 수 없습니다. 취소하려면 반대 값으로 다시 투표하세요.
- ⚠️ `value`에 쓸 수 있는 값은 스펙에서 확인하지 못했습니다. 예시에는 `1`(좋아요)만 나옵니다.

✅ 실제 응답 (`POST`는 201, `GET /votes?sub_id=`는 200):
```json
{ "id": 2625759, "image_id": "KWdLHmOqc", "sub_id": "my-user-123",
  "created_at": "2026-11-09T02:32:50.291Z", "value": 1, "country_code": "KR" }
```

---

### 4-5. 즐겨찾기 (Favourites) — 키 필요

| 메서드 | 경로 | 설명 |
|---|---|---|
| GET | `/favourites` | 목록. `sub_id`, `attach_image=true`(이미지 정보 포함) |
| GET | `/favourites/{id}` | 하나 조회 |
| POST | `/favourites` | 추가. body: `{"image_id": "...", "sub_id": "..."}` |
| DELETE | `/favourites/{id}` | 삭제 |

```js
// 추가
await fetch("https://api.thecatapi.com/v1/favourites", {
  method: "POST",
  headers: { "Content-Type": "application/json", "x-api-key": API_KEY },
  body: JSON.stringify({ image_id: "KWdLHmOqc", sub_id: "my-user-123" }),
});

// 삭제
await fetch(`https://api.thecatapi.com/v1/favourites/${favId}`, {
  method: "DELETE",
  headers: { "x-api-key": API_KEY },
});
```

✅ 실제 응답:
- `POST`: **201**과 `{"id":1540748,"image_id":"KWdLHmOqc","sub_id":"my-user-123","created_at":"..."}`이 옵니다. 삭제할 때는 이 `id`를 씁니다.
- `GET /favourites?sub_id=`: `attach_image`를 넣지 않아도 `image` 객체(`id`, `url`, `width`, `height`, `breeds` 등)가 함께 옵니다.
- `DELETE`: 200과 함께 빈 객체 `{}`가 옵니다.

---

### 4-6. 고양이 상식 (Facts) — 키 필요, Startup 이상

✅ Free 요금제 키로 `GET /facts/random`을 호출하면 **403**이 옵니다. 무료로 상식을 보여주려면 다른 API가 필요합니다.

| 경로 | 설명 |
|---|---|
| `GET /facts` | 목록. `limit`, `offset`, `breed_id`, `category`, `lang` |
| `GET /facts/random` | 무작위 하나 |
| `GET /facts/breed/{breedId}` | 품종별 목록 |
| `GET /facts/breed/{breedId}/random` | 품종별 무작위 하나 |
| `GET /facts/{id}` | 하나 조회 |

응답 예시:
```json
{ "id": "aB3xY9mK12", "title": "...", "fact": "...", "category": "fun", "species_id": 1 }
```

---

### 4-7. 기타
- `GET /sub-ids`: sub_id별 이미지·투표·즐겨찾기 수
- `GET /accounts/quota`, `GET /accounts/features`: 계정 한도와 켜진 기능
- Health tips, Pets, Webhooks 엔드포인트도 있습니다 (Enterprise 위주).

---

## 5. 웹페이지에서 쓸 때 참고할 점

1. **`sub_id`로 사용자 구분**: 로그인 기능이 없어도 `sub_id`에 임의의 사용자 ID를 넣으면 투표와 즐겨찾기를 사용자별로 나눠 관리할 수 있습니다.
2. **API 키 노출**: 프론트엔드 JS에 키를 넣으면 페이지 소스에서 누구나 볼 수 있습니다. 학습용이라면 괜찮지만, 실제 서비스라면 서버나 서버리스 함수를 프록시로 두세요.
3. **CORS**: ✅ 응답 헤더가 `Access-Control-Allow-Origin: *`이라서 브라우저에서 `fetch`로 바로 호출할 수 있습니다.
4. **Free 요금제 한도**: 분당 10회입니다. 이미지를 한 번에 여러 장(`limit=10`) 받아 두면 호출 수를 줄일 수 있습니다.
5. **페이지네이션**: `limit`과 `page`(0부터 시작)를 조합해 씁니다.
6. **API 키 보관**: 키는 `.env`의 `CAT_KEY`에 있습니다. 브라우저는 `.env`를 읽을 수 없으므로 JS에서 쓸 방법을 따로 정해야 합니다(예: `config.js`로 분리하고 `.gitignore`에 추가).

---

## 7. 실제 테스트 결과 (2026-11-09, Free 요금제 키)

| 엔드포인트 | 결과 | 비고 |
|---|---|---|
| `GET /images/search` (키 없이) | ✅ 200 | `Access-Control-Allow-Origin: *` |
| `GET /accounts/features` | ✅ 200 | `[]` (유료 기능 없음) |
| `GET /breeds?limit=3` | ✅ 200 | 점수 필드 없음. `weight`, `height` 있음 |
| `GET /breeds/search?q=siam` | ✅ 200 | 3개 |
| `GET /breeds/abys` | ✅ 200 | |
| `GET /images/search?breed_ids=beng&limit=2` | ✅ 200 | 이미지마다 `breeds` 배열 포함 |
| `GET /categories` | ✅ 200 | `[]` |
| `GET /facts/random` | ❌ 403 | Startup 이상 필요 |
| `POST /favourites` | ✅ 201 | `id` 반환 |
| `GET /favourites?sub_id=` | ✅ 200 | `image` 객체 포함 |
| `DELETE /favourites/{id}` | ✅ 200 | `{}` |
| `POST /votes` | ✅ 201 | 삭제 불가 |
| `GET /votes?sub_id=` | ✅ 200 | |

> 테스트용 투표 1건이 `sub_id: claude-test`로 계정에 남아 있습니다. 실제 페이지에서는 다른 `sub_id`를 쓰세요.

---

## 8. 참고 링크
- 문서: https://docs.thecatapi.com/
- API 레퍼런스: https://docs.thecatapi.com/reference
- OpenAPI 스펙: https://api.thecatapi.com/openapi-json
- 투표 데모: https://examples.thecatapi.com/voting/
- 품종 검색 데모: https://examples.thecatapi.com/breed-search/
