---
name: law
description: 한국 법령 조회, 검색, 조문 비교, 개정 감지, 의무항목 매핑. 국가법령정보센터 Open API 사용. Korean law lookup, search, diff, amendment detection, obligation mapping via law.go.kr API.
argument-hint: "[search|article|diff|recent|history|map] [arguments]"
disable-model-invocation: true
allowed-tools: "Bash(curl *), Bash(node *), Bash(cat *), Bash(psql *), Bash(docker *), Read, Write, Grep"
---

# /law — 한국 법령 조회 스킬

국가법령정보센터 공동활용 Open API를 사용하여 한국 법령을 조회·검색·비교합니다.

## 동작 모드

이 스킬은 두 가지 모드로 동작합니다:

- **범용 모드** (기본) — curl만으로 동작. DB, 프레임워크, 런타임 의존성 없음.
  `search`, `article`, `diff`, `recent`, `history` 서브커맨드 사용 가능
- **DB 모드** (선택) — PostgreSQL + Prisma 기반 프로젝트에서 추가 기능 제공.
  `map` 서브커맨드 사용 가능. `article`, `diff`에서 의무항목 매핑·참조 법령 자동 표시

**DB 모드 자동 감지:** 프로젝트에 `prisma/schema.prisma` 파일이 있고 `Regulation` 모델이 정의되어 있으면 DB 모드가 활성화됩니다.

## 환경 설정

이 스킬은 **OC 값**(API 인증키)이 필요합니다. 아래 순서로 확인합니다:

1. 환경변수 `LAW_API_OC_KEY`
2. `.env` 파일의 `LAW_API_OC_KEY`
3. 둘 다 없으면 사용자에게 안내:
   ```
   LAW_API_OC_KEY가 설정되지 않았습니다.
   https://open.law.go.kr 에서 회원가입 후 이메일 ID를 OC 값으로 사용하세요.
   설정: export LAW_API_OC_KEY="your@email.com"
   ```

OC 값 확인:
!`echo "${LAW_API_OC_KEY:-$(grep -s LAW_API_OC_KEY .env 2>/dev/null | cut -d'"' -f2 | head -1)}" | head -c 50`

DB 모드 확인:
!`test -f prisma/schema.prisma && grep -q "model Regulation" prisma/schema.prisma 2>/dev/null && echo "DB_MODE=ON" || echo "DB_MODE=OFF"`

## API 기본 정보

| 항목 | 값 |
|------|-----|
| 검색 | `http://www.law.go.kr/DRF/lawSearch.do?OC={OC}&target=law&type=JSON&query={검색어}&display=100` |
| 본문 | `http://www.law.go.kr/DRF/lawService.do?OC={OC}&target=law&type=JSON&ID={법령ID}` |
| 이력 | `http://www.law.go.kr/DRF/lawSearch.do?OC={OC}&target=lsHstInf&type=JSON&regDt={YYYYMMDD}&display=100` |
| 인증 | 쿼리 파라미터 `OC` = 등록된 이메일 ID |
| 응답 | JSON |
| 속도제한 | 명시적 제한 없음 (1초 간격 권장) |

## 서브커맨드

인자(`$ARGUMENTS`)에 따라 아래와 같이 분기합니다.

---

### 1. `/law search <키워드>` — 법령 검색

키워드로 법령을 검색합니다.

**실행:**
```bash
curl -s -m 30 "http://www.law.go.kr/DRF/lawSearch.do?OC={OC}&target=law&type=JSON&query={키워드}&display=20"
```

**응답 파싱:** `LawSearch.law` 배열에서 추출

**출력 형식:**
```
검색 결과: "{키워드}" (총 N건)

 # │ 법령명                        │ 구분    │ 제개정   │ 시행일
───┼──────────────────────────────┼────────┼─────────┼───────────
 1 │ 산업안전보건법                  │ 법률    │ 일부개정 │ 2024-01-01
 2 │ 산업안전보건법 시행령            │ 대통령령 │ 일부개정 │ 2024-03-01

조문 조회: /law article 산업안전보건법 제29조
```

**주의:**
- `LawSearch.law`가 배열이 아닌 단일 객체일 수 있음 → 배열로 감싸서 처리
- `totalCnt`가 문자열로 옴 → 숫자 변환

---

### 2. `/law article <법령명> [조항]` — 조문 조회

법령 본문을 조회합니다.

**실행 순서:**

1. 법령 검색으로 `법령ID` 확보:
```bash
curl -s -m 30 "http://www.law.go.kr/DRF/lawSearch.do?OC={OC}&target=law&type=JSON&query={법령명}&display=10"
```

2. `법령명한글`이 정확히 일치하는 항목의 `법령ID` 사용 (없으면 첫 번째)

3. 본문 조회:
```bash
curl -s -m 30 "http://www.law.go.kr/DRF/lawService.do?OC={OC}&target=law&type=JSON&ID={법령ID}"
```

4. 응답 파싱:
   - `법령.조문.조문단위` 배열에서 `조문여부 === "조문"` 필터
   - `조문번호`로 요청 조항 매칭 (예: "제4조")
   - 조항 미지정이면 전체 조문 목록 표시

5. 조문 내용 조립 (계층 구조):
   - `조문내용` (본문)
   - `항[]` → `항내용` (①, ②, ③...)
   - `항[].호[]` → `호내용` (1., 2., 3....)
   - 항/호가 단일 객체일 수 있으므로 배열로 통일

6. **[DB 모드]** 연결된 의무항목·참조 법령 함께 조회 (DB 접근 가능한 경우):
   - `obligation_items` 테이블에서 해당 `regulationId`로 매핑된 의무항목 조회
   - `regulation_cross_references` 테이블에서 참조 관계 조회

**출력 형식 (범용 모드):**
```
━━ {법령명} {조문번호} ━━
{조문제목}

{조문내용}
  ① {항내용}
    1. {호내용}
    2. {호내용}
  ② {항내용}

시행일: {조문시행일자} | 법령구분: {법령구분명}
소관: {소관부처명}
```

**출력 형식 (DB 모드 — 의무항목 매핑 추가):**
```
━━ {법령명} {조문번호} ━━
{조문제목}

{조문내용}
  ① {항내용}
  ② {항내용}

┌─ 연결된 의무항목 ──────────────────────┐
│ O-01  안전보건 목표·경영방침 설정  (연간) │
│ O-02  전담조직 설치              (연간) │
│ O-03  유해·위험요인 확인·개선    (반기) │
└────────────────────────────────────────┘

┌─ 참조 법령 ────────────────────────────┐
│ → 시행령 제4조 (안전보건관리체계 구축)    │
│ → 산업안전보건법 제15조                  │
└────────────────────────────────────────┘

시행일: 2022-01-27 | 최종 개정: 2024-01-01
```

조항 미지정 시:
```
━━ {법령명} — 조문 목록 ━━

  제1조  (목적)
  제2조  (정의)
  제3조  (적용범위)
  제4조  (사업주와 경영책임자등의 안전 및 보건 확보의무)
  ...

조문 조회: /law article {법령명} 제4조
```

---

### 3. `/law diff <법령명> <조항> [비교대상]` — 조문 비교

API 최신 조문 내용과 비교대상을 비교합니다.

**비교대상 (우선순위):**
1. 파일 경로가 주어지면 → 해당 파일의 내용과 비교
2. [DB 모드] 비교대상 없으면 → DB `regulations` 테이블에서 해당 조문 조회하여 비교
3. 비교 대상이 없으면 → API 조문 내용만 표시하고 "비교 대상을 지정해주세요" 안내

**실행:**

1. API에서 최신 조문 조회 (article 서브커맨드와 동일)
2. 비교 대상 확보 (파일 읽기 or DB 조회)
3. 공백 정규화 후 비교: `text.replace(/\s+/g, " ").trim()`

**출력 형식:**
```
━━ {법령명} {조문번호} 비교 ━━

상태: ⚠️ 차이 발견

[변경 전]
  ① 사업주 또는 경영책임자등은...

[변경 후 (API 최신)]
  ① 사업주 또는 경영책임자등은... (변경된 부분)

변경 유형: 일부개정
API 시행일: 2024-01-01
```

일치 시:
```
상태: ✅ 동일 (변경 없음)
API 시행일: 2024-01-01
```

---

### 4. `/law recent [법령명] [일수]` — 최근 개정 조회

최근 개정된 법령을 조회합니다.

**인자:**
- `법령명`: 특정 법령만 필터 (선택)
- `일수`: 최근 N일 (기본 30일)

**실행:**

최근 N일간 날짜별로 변경이력 API를 호출합니다 (최대 7일씩 조회):

```bash
curl -s -m 30 "http://www.law.go.kr/DRF/lawSearch.do?OC={OC}&target=lsHstInf&type=JSON&regDt={YYYYMMDD}&display=100"
```

**주의:** 날짜별 호출이므로 30일이면 최대 30번 호출 → `sleep 1` 삽입

법령명 필터가 있으면 결과에서 `법령명한글`로 필터링합니다.

**출력 형식:**
```
━━ 최근 법령 개정 현황 (30일) ━━

날짜       │ 법령명                        │ 구분    │ 제개정
───────────┼──────────────────────────────┼────────┼─────────
2024-12-15 │ 산업안전보건법                  │ 법률    │ 일부개정
2024-12-10 │ 중대재해 처벌 등에 관한 법률     │ 법률    │ 일부개정

총 2건의 개정이 있었습니다.
```

개정이 없을 때:
```
최근 30일간 개정된 법령이 없습니다.
마지막 확인: 2024-12-20
```

---

### 5. `/law history <법령명>` — 법령 연혁

특정 법령의 연혁(제정·개정 이력)을 조회합니다.

**실행:**
```bash
curl -s -m 30 "http://www.law.go.kr/DRF/lawSearch.do?OC={OC}&target=law&type=JSON&query={법령명}&display=100"
```

검색 결과에서 동일 법령의 `제개정구분명`, `공포일자`, `시행일자`를 시간순 정렬합니다.

**출력 형식:**
```
━━ {법령명} 연혁 ━━

공포일       │ 시행일       │ 구분
─────────────┼─────────────┼──────────
2022-01-27   │ 2022-01-27   │ 제정
2023-05-16   │ 2024-01-01   │ 일부개정
2024-01-02   │ 2024-07-01   │ 일부개정

현행: 시행 2024-07-01 (2024-01-02 일부개정)
```

---

### 6. `/law map <O-코드>` — 의무항목 매핑 (DB 모드 전용)

**[DB 모드 전용]** 의무항목 코드에서 근거 법령을 역추적합니다.

DB 모드가 아닌 경우:
```
/law map은 DB 모드에서만 사용 가능합니다.
프로젝트에 prisma/schema.prisma가 있고 Regulation 모델이 정의되어야 합니다.
```

**DB 접근 방법 (프로젝트에 맞게 선택):**
- `npx prisma db execute` 또는 `psql` 직접 실행
- Docker 환경이면 `docker exec {container} psql ...`
- `.env`의 `DATABASE_URL`에서 접속 정보 확인

**조회 SQL:**

1. 의무항목 + 연결 법령:
```sql
SELECT
  oi.code, oi.title, oi.description, oi.period,
  oi."evidenceGuide",
  r."lawName", r."articleNumber", r."articleTitle", r.content
FROM obligation_items oi
LEFT JOIN regulations r ON oi."regulationId" = r.id
WHERE oi.code = '{O-코드}';
```

2. 참조·위임 관계:
```sql
SELECT
  rcr.type, rcr.description, rcr."articleClause",
  t."lawName", t."articleNumber", t."articleTitle"
FROM regulation_cross_references rcr
JOIN regulations t ON rcr."targetId" = t.id
JOIN regulations s ON rcr."sourceId" = s.id
JOIN obligation_items oi ON oi."regulationId" = s.id
WHERE oi.code = '{O-코드}';
```

**출력 형식:**
```
━━ O-01: 안전보건 목표·경영방침 설정 ━━

이행주기: 연간
근거법령: 중대재해처벌법 제4조 제1호
         "사업 또는 사업장의 안전·보건에 관한 목표와
          경영방침을 설정할 것"

증빙자료 가이드:
  - 안전보건 경영방침 문서
  - 이사회/경영진 승인 기록
  - 종사자 공유/게시 증빙

┌─ 참조·위임 관계 ──────────────────────┐
│ 위임 → 시행령 제4조 (세부 기준)         │
│ 참조 → 산업안전보건법 제15조            │
└────────────────────────────────────────┘
```

---

### 인자 없이 `/law` 호출 시

사용법을 안내합니다:

```
━━ /law — 한국 법령 조회 도구 ━━

  /law search <키워드>              법령 검색
  /law article <법령명> [조항]       조문 조회
  /law diff <법령명> <조항> [비교]   API vs 로컬 비교
  /law recent [법령명] [일수]        최근 개정 조회
  /law history <법령명>             법령 연혁
  /law map <O-코드>                의무항목 매핑 (DB 모드)

예시:
  /law search 안전보건교육
  /law article 중대재해처벌법 제4조
  /law diff 산업안전보건법 제29조 ./docs/old.txt
  /law recent 중대재해처벌법 90
  /law history 산업안전보건법
  /law map O-01

설정: LAW_API_OC_KEY 환경변수에 OC 값 설정
발급: https://open.law.go.kr
```

## API 응답 구조 참고

### 검색 응답 (lawSearch.do)
```json
{
  "LawSearch": {
    "totalCnt": "3",
    "page": "1",
    "law": [
      {
        "법령일련번호": "...",
        "법령명한글": "산업안전보건법",
        "법령약칭명": "산안법",
        "법령ID": 12345,
        "공포일자": "20240101",
        "시행일자": "20240301",
        "제개정구분명": "일부개정",
        "소관부처명": "고용노동부",
        "법령구분명": "법률",
        "법령상세링크": "/법령/산업안전보건법"
      }
    ]
  }
}
```

### 본문 응답 (lawService.do)
```json
{
  "법령": {
    "기본정보": {
      "법령ID": 12345,
      "법령명_한글": "산업안전보건법",
      "시행일자": "20240301",
      "소관부처명": "고용노동부",
      "법령구분명": "법률",
      "제개정구분명": "일부개정"
    },
    "조문": {
      "조문단위": [
        {
          "조문번호": "제1조",
          "조문여부": "조문",
          "조문제목": "목적",
          "조문시행일자": "20240301",
          "조문내용": "이 법은...",
          "항": [
            {
              "항번호": "1",
              "항내용": "① 사업주는...",
              "호": [
                {
                  "호번호": "1",
                  "호내용": "1. 안전보건..."
                }
              ]
            }
          ]
        }
      ]
    }
  }
}
```

### 핵심 파싱 주의사항

1. **단일 객체 vs 배열**: `law`, `조문단위`, `항`, `호` 모두 결과가 1개면 객체, 2개 이상이면 배열로 옴.
   항상 배열로 통일: `const arr = Array.isArray(v) ? v : v ? [v] : []`

2. **조문여부 필터**: `조문여부 === "조문"`만 처리. "편", "장", "절" 등은 구조 구분자.

3. **숫자가 문자열**: `totalCnt`, `법령ID` 등이 문자열로 올 수 있음.

4. **날짜 형식**: `YYYYMMDD` 문자열. 표시할 때 `YYYY-MM-DD`로 변환.

## 에러 처리

| 상황 | 대응 |
|------|------|
| OC 미설정 | 발급 안내 출력, API 호출 중단 |
| API 응답 없음/타임아웃 | "국가법령정보센터 응답 없음. 잠시 후 재시도해주세요." |
| 법령 검색 결과 0건 | 유사 키워드 제안 (띄어쓰기 변형 등) |
| 조문 번호 불일치 | 해당 법령의 전체 조문 목록 표시 |
| JSON 파싱 실패 | 원본 응답 일부 표시 + 에러 보고 |
| DB 연결 실패 (DB 모드) | 범용 모드로 폴백, DB 접속 정보 확인 안내 |

## curl 실행 규칙

- 항상 `-s` (silent) + `-m 30` (30초 타임아웃) 플래그 사용
- 연속 호출 시 `sleep 1` 삽입 (rate-limit 대응)
- URL의 한글 파라미터는 그대로 전달 (curl이 자동 인코딩)
- 응답이 길면 `jq`로 필요한 필드만 추출 (jq 없으면 node -e로 대체)
