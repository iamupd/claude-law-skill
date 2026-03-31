<p align="center">
  <img src="https://img.shields.io/badge/Claude_Code-Skill-blueviolet?style=for-the-badge" alt="Claude Code Skill" />
  <img src="https://img.shields.io/badge/API-국가법령정보센터-007aff?style=for-the-badge" alt="Law API" />
  <img src="https://img.shields.io/badge/lang-한국어-red?style=for-the-badge" alt="Korean" />
</p>

<p align="center">
  <img src="https://img.shields.io/badge/license-MIT-green?style=flat-square" alt="License" />
  <img src="https://img.shields.io/badge/PRs-welcome-ff69b4?style=flat-square" alt="PRs Welcome" />
</p>

# /law - 한국 법령 조회 스킬 (Claude Code)

[국가법령정보센터 Open API](https://open.law.go.kr)를 활용하여 한국 법령을 조회하는 **Claude Code 스킬**입니다.

개발 중 법령 내용을 빠르게 참조하고, DB에 저장된 의무항목과의 매핑 관계까지 확인할 수 있습니다.

## 사용법

```
/law 중대재해처벌법 제4조           # 특정 조문 조회
/law search 안전보건교육            # 키워드 검색
/law map O-01                      # 의무항목 → 근거법령 역추적
/law diff 제4조                    # DB vs API 최신 내용 비교
/law recent                        # 최근 개정 법령 조회
```

## 출력 예시

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

## 설치

`law.md` 파일을 프로젝트에 추가하고 `CLAUDE.md`에서 참조합니다.

```bash
# 다운로드
mkdir -p skills/law
curl -o skills/law/law.md https://raw.githubusercontent.com/iamupd/claude-law-skill/main/law.md
```

`CLAUDE.md`에 추가:

```markdown
## Custom Skills
- `/law` — 한국 법령 조회. 정의: `skills/law/law.md`
```

## 기능 상세

### 조문 조회 (`/law <법령명> <조항>`)

국가법령정보센터 API로 조문 내용을 조회합니다. DB에 의무항목이 매핑되어 있으면 함께 표시합니다.

### 키워드 검색 (`/law search <키워드>`)

법령명, 조문 제목 등에서 키워드를 검색하여 테이블로 출력합니다.

### 의무항목 매핑 (`/law map <O-코드>`)

의무항목 코드(O-01~O-13)로 근거 법령을 역추적합니다. **API 키 없이 DB만으로 동작합니다.**

### DB vs API 비교 (`/law diff <조항>`)

DB에 저장된 법령 내용과 API 최신 내용을 비교하여 개정 여부를 확인합니다.

### 최근 개정 (`/law recent`)

관리 대상 법령의 최근 개정 현황을 조회합니다.

## API 키 없이 사용

| 커맨드 | API 키 필요 |
|--------|:-----------:|
| `/law map <O-코드>` | X |
| `/law diff` (DB 내용만) | X |
| `/law <법령명> <조항>` | O |
| `/law search <키워드>` | O |
| `/law recent` | O |

API 키 발급: [open.law.go.kr](https://open.law.go.kr) 가입 후 `.env`에 `LAW_API_OC_KEY="이메일ID"` 추가

## 대상 법령

- 중대재해 처벌 등에 관한 법률 (+ 시행령)
- 산업안전보건법 (+ 시행령)

## 국가법령정보센터 API

| 항목 | 값 |
|------|-----|
| 검색 URL | `http://www.law.go.kr/DRF/lawSearch.do` |
| 본문 URL | `http://www.law.go.kr/DRF/lawService.do` |
| 인증 | 쿼리 파라미터 `OC={이메일ID}` |
| 응답 형식 | JSON |
| Rate Limit | 없음 (1초 딜레이 권장) |

## 라이선스

MIT
