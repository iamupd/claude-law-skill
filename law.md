# Korean Law Lookup Skill for Claude Code

국가법령정보센터 Open API를 통해 한국 법령을 조회하고, 프로젝트 DB의 의무항목 매핑 정보와 함께 표시하는 Claude Code 스킬입니다.

## Trigger

사용자가 `/law` 명령을 사용하거나, 한국 법령 조회/검색을 요청할 때 실행합니다.

## Arguments

다음 서브커맨드를 지원합니다:

- `/law <법령명> <조항>` — 특정 조문 조회 (예: `/law 중대재해처벌법 제4조`)
- `/law search <키워드>` — 키워드로 법령 검색 (예: `/law search 안전보건교육`)
- `/law map <O-코드>` — 의무항목의 근거 법령 역추적 (예: `/law map O-01`)
- `/law diff <조항>` — API 최신 내용 vs DB 저장 내용 비교
- `/law recent` — 최근 개정 법령 조회

인자 없이 `/law`만 호출하면 사용법을 안내합니다.

## Instructions

### 환경 확인

1. 프로젝트 루트에서 `.env` 파일의 `LAW_API_OC_KEY` 값을 확인합니다.
2. API 키가 없으면 DB 조회 기반 기능(`map`, `diff`)만 사용 가능함을 안내합니다.
3. API 키 발급: https://open.law.go.kr 에서 회원가입 후 발급

### 서브커맨드별 실행 방법

#### 1. 조문 조회: `/law <법령명> <조항>`

**API 키 필요**

국가법령정보센터 API를 호출하여 조문을 조회합니다.

**실행 순서:**

1. `src/lib/law-sync/api-client.ts`의 `searchLaw()` 함수 로직을 참고하여 법령을 검색합니다.

```
API URL: http://www.law.go.kr/DRF/lawSearch.do?OC={API_KEY}&target=law&type=JSON&query={법령명}&display=20
```

2. 검색 결과에서 정확히 일치하는 법령의 `법령ID`를 찾습니다.

3. `getLawFullText()` 로직으로 본문을 조회합니다.

```
API URL: http://www.law.go.kr/DRF/lawService.do?OC={API_KEY}&target=law&type=JSON&ID={법령ID}
```

4. 응답에서 해당 조항(예: "제4조")을 찾아 파싱합니다.
   - `법령.조문.조문단위` 배열에서 `조문번호`가 일치하는 항목
   - 조문 내용 + 항(항번호, 항내용) + 호(호번호, 호내용)를 계층적으로 조립

5. **DB 매핑 정보도 함께 조회합니다:**

```bash
# Prisma로 연결된 의무항목 조회
docker exec safeshield-db psql -U safeshield_user -d safeshield -c "
  SELECT oi.code, oi.title, oi.period
  FROM obligation_items oi
  JOIN regulations r ON oi.\"regulationId\" = r.id
  WHERE r.\"articleNumber\" = '제4조'
    AND r.\"lawName\" LIKE '%중대재해%';
"
```

6. **출력 형식:**

```
━━ {법령명} {조항번호} ━━
{조문제목}

{조문내용}
  ① {항내용}
    1. {호내용}
    2. {호내용}
  ② {항내용}
  ...

┌─ 연결된 의무항목 ──────────────────────┐
│ O-01  안전보건 목표·경영방침 설정  (연간) │
│ O-02  전담조직 설치              (연간) │
│ O-03  유해·위험요인 확인·개선    (반기) │
│ ...                                    │
└────────────────────────────────────────┘

┌─ 참조 법령 ────────────────────────────┐
│ → 시행령 제4조 (안전보건관리체계 구축)    │
│ → 산업안전보건법 제15조                  │
└────────────────────────────────────────┘

시행일: 2022-01-27 | 최종 개정: 2024-01-01
```

#### 2. 키워드 검색: `/law search <키워드>`

**API 키 필요**

1. 국가법령정보센터 검색 API를 호출합니다.

```
API URL: http://www.law.go.kr/DRF/lawSearch.do?OC={API_KEY}&target=law&type=JSON&query={키워드}&display=20
```

2. 결과를 테이블로 정리합니다:

```
검색 결과: "{키워드}" (총 N건)

 # │ 법령명                              │ 구분   │ 시행일
───┼────────────────────────────────────┼───────┼───────────
 1 │ 산업안전보건법                       │ 법률   │ 2024-01-01
 2 │ 산업안전보건법 시행령                 │ 대통령령│ 2024-03-01
 3 │ 산업안전보건법 시행규칙               │ 부령   │ 2024-03-01

상세 조회: /law 산업안전보건법 제29조
```

#### 3. 의무항목 매핑: `/law map <O-코드>`

**API 키 불필요 (DB 조회)**

의무항목 코드에서 근거 법령을 역추적합니다.

1. DB에서 의무항목과 연결된 법령을 조회합니다:

```bash
docker exec safeshield-db psql -U safeshield_user -d safeshield -c "
  SELECT
    oi.code, oi.title, oi.description, oi.period,
    oi.\"evidenceGuide\",
    r.\"lawName\", r.\"articleNumber\", r.\"articleTitle\", r.content
  FROM obligation_items oi
  LEFT JOIN regulations r ON oi.\"regulationId\" = r.id
  WHERE oi.code = 'O-01';
"
```

2. 참조 관계도 함께 조회합니다:

```bash
docker exec safeshield-db psql -U safeshield_user -d safeshield -c "
  SELECT
    rcr.type, rcr.description, rcr.\"articleClause\",
    t.\"lawName\", t.\"articleNumber\", t.\"articleTitle\"
  FROM regulation_cross_references rcr
  JOIN regulations t ON rcr.\"targetId\" = t.id
  JOIN regulations s ON rcr.\"sourceId\" = s.id
  JOIN obligation_items oi ON oi.\"regulationId\" = s.id
  WHERE oi.code = 'O-01';
"
```

3. **출력 형식:**

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

#### 4. DB vs API 비교: `/law diff <조항>`

**API 키 필요**

1. DB에서 해당 조항의 현재 내용을 조회합니다.
2. API에서 최신 내용을 가져옵니다.
3. 두 내용을 비교하여 차이점을 표시합니다.
4. `src/lib/law-sync/differ.ts`의 `normalizeContent()` 로직으로 공백 차이는 무시합니다.

```
━━ 중대재해처벌법 제4조 비교 결과 ━━

상태: ✅ 동일 (변경 없음)
DB 최종 업데이트: 2024-01-01
API 시행일: 2024-01-01
```

또는:

```
상태: ⚠️ 차이 발견

- (삭제) 제3항: 기존 내용...
+ (추가) 제3항: 변경된 내용...

→ /api/regulations/sync 에서 동기화를 실행하세요.
```

#### 5. 최근 개정: `/law recent`

**API 키 필요**

1. 국가법령정보센터의 변경이력 API를 호출합니다.
2. 관리 대상 법령(중대재해처벌법, 산업안전보건법)의 최근 변경사항만 필터링합니다.

```
━━ 최근 법령 개정 현황 ━━

최근 30일 이내 변경 없음.

마지막 개정:
  중대재해 처벌 등에 관한 법률 — 2024-01-01 일부개정
  산업안전보건법 — 2024-03-01 일부개정
```

### API 키 없을 때 동작

API 키가 설정되지 않은 경우:

```
⚠️ LAW_API_OC_KEY가 설정되지 않았습니다.

사용 가능한 기능:
  /law map <O-코드>  — DB 기반 의무항목 매핑 조회
  /law diff <조항>   — DB 저장 내용 확인 (API 비교 불가)

API 조회 기능을 사용하려면:
  1. https://open.law.go.kr 에서 API 키 발급
  2. .env 파일에 LAW_API_OC_KEY="발급받은_이메일" 추가
```

### 에러 처리

- API 타임아웃 (30초): "국가법령정보센터 API 응답 시간 초과" 안내
- 법령을 찾을 수 없음: 유사한 법령명 제안
- 조항을 찾을 수 없음: 해당 법령의 전체 조항 목록 표시
- DB 연결 실패: Docker 컨테이너 상태 확인 안내

### 참고 파일

이 스킬이 활용하는 프로젝트 내 코드:
- `src/lib/law-sync/api-client.ts` — API 호출 로직 (URL, 인증, 타임아웃)
- `src/lib/law-sync/parser.ts` — 응답 파싱 (조문 계층 구조)
- `src/lib/law-sync/types.ts` — 타입 정의
- `src/lib/law-sync/differ.ts` — 내용 비교 알고리즘
- `prisma/schema.prisma` — Regulation, ObligationItem 스키마
