# claude-law-skill

![Claude Skill](https://img.shields.io/badge/Claude-Cowork%20Skill-blueviolet?logo=anthropic&logoColor=white)
![API](https://img.shields.io/badge/API-국가법령정보센터-007aff)
![Lang](https://img.shields.io/badge/lang-한국어-red)
![License](https://img.shields.io/badge/License-MIT-green)
![PRs](https://img.shields.io/badge/PRs-welcome-ff69b4)

> Claude Cowork 스킬 — 한국 법령 조회 및 의무항목 매핑

[국가법령정보센터 Open API](https://open.law.go.kr)를 활용하여 한국 법령을 조회하는 Claude Code 스킬입니다. 개발 중 법령 내용을 빠르게 참조하고, DB에 저장된 의무항목과의 매핑 관계까지 확인할 수 있습니다.

---

## 설치

`law.md` 파일을 프로젝트에 추가하고 `CLAUDE.md`에서 참조합니다.

```bash
mkdir -p skills/law
curl -o skills/law/law.md https://raw.githubusercontent.com/iamupd/claude-law-skill/main/law.md
```

`CLAUDE.md`에 추가:

```markdown
## Custom Skills
- `/law` — 한국 법령 조회. 정의: `skills/law/law.md`
```

---

## 사용법

### Claude 스킬로 사용

Claude에게 슬래시 명령으로 요청합니다:

```
/law 중대재해처벌법 제4조           # 특정 조문 조회
/law search 안전보건교육            # 키워드 검색
/law map O-01                      # 의무항목 → 근거법령 역추적
/law diff 제4조                    # DB vs API 최신 내용 비교
/law recent                        # 최근 개정 법령 조회
```

### 출력 예시

```
━━ 중대재해 처벌 등에 관한 법률 제4조 ━━
사업주와 경영책임자등의 안전 및 보건 확보의무

사업주 또는 경영책임자등은 사업주나 법인 또는 기관이 실질적으로
지배·운영·관리하는 사업 또는 사업장에서 종사자의 안전·보건상 ...

  ① 재해예방에 필요한 인력 및 예산 등 안전보건관리체계의 구축 ...
  ② ...

┌─ 연결된 의무항목 ──────────────────────┐
│ O-01  안전보건 목표·경영방침 설정  (연간) │
│ O-02  전담조직 설치              (연간) │
│ O-03  유해·위험요인 확인·개선    (반기) │
│ ...                                    │
└────────────────────────────────────────┘

시행일: 2022-01-27 | 최종 개정: 2024-01-01
```

---

## 구조

```
claude-law-skill/
├── law.md        # Claude 스킬 정의 (트리거 조건 + 실행 지침)
├── README.md     # 이 문서
└── LICENSE       # MIT
```

---

## 기능 상세

| 커맨드 | 설명 | API 키 |
|--------|------|:------:|
| `/law <법령명> <조항>` | 조문 조회 + 의무항목 매핑 + 참조 법령 | 필요 |
| `/law search <키워드>` | 법령명·조문 키워드 검색 | 필요 |
| `/law map <O-코드>` | 의무항목 코드로 근거법령 역추적 | 불필요 |
| `/law diff <조항>` | DB 저장 내용 vs API 최신 비교 | 필요 |
| `/law recent` | 관리 대상 법령 최근 개정 현황 | 필요 |

**API 키 없이 사용 가능한 기능:** `map`, `diff`(DB 내용만)

---

## 대상 법령

| 법령 | 구분 |
|------|------|
| 중대재해 처벌 등에 관한 법률 | 법률 |
| 중대재해 처벌 등에 관한 법률 시행령 | 대통령령 |
| 산업안전보건법 | 법률 |
| 산업안전보건법 시행령 | 대통령령 |

---

## API 키 발급

1. [open.law.go.kr](https://open.law.go.kr) 회원가입
2. API 키 발급 신청
3. `.env`에 설정:

```env
LAW_API_OC_KEY="등록된_이메일_ID"
```

---

## 국가법령정보센터 API

| 항목 | 값 |
|------|-----|
| 검색 URL | `http://www.law.go.kr/DRF/lawSearch.do` |
| 본문 URL | `http://www.law.go.kr/DRF/lawService.do` |
| 인증 | 쿼리 파라미터 `OC={이메일ID}` |
| 응답 형식 | JSON (`type=JSON`) |
| Rate Limit | 없음 (1초 딜레이 권장) |

---

## 사전 요구사항

| 기능 | 필요 환경 |
|------|----------|
| 조문 조회, 검색, 최근 개정 | 국가법령정보센터 API 키 |
| 의무항목 매핑 (`map`) | PostgreSQL + Prisma (`Regulation`, `ObligationItem` 모델) |
| DB 비교 (`diff`) | PostgreSQL + Prisma |

---

## 한계

- 국가법령정보센터 API가 일시적으로 불안정할 수 있습니다 (타임아웃 30초)
- 부칙은 조회 대상에서 제외됩니다
- DB 매핑 기능(`map`, `diff`)은 Prisma 기반 프로젝트에서만 동작합니다

---

## 라이선스

![License](https://img.shields.io/badge/License-MIT-green)

MIT
