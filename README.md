# claude-law-skill

![Claude Skill](https://img.shields.io/badge/Claude-Cowork%20Skill-blueviolet?logo=anthropic&logoColor=white)
![Plugin](https://img.shields.io/badge/Claude-Plugin-blue?logo=anthropic&logoColor=white)
![API](https://img.shields.io/badge/API-국가법령정보센터-007aff)
![Dependencies](https://img.shields.io/badge/의존성-없음-brightgreen)
![Lang](https://img.shields.io/badge/lang-한국어-red)
![License](https://img.shields.io/badge/License-MIT-green)
![PRs](https://img.shields.io/badge/PRs-welcome-ff69b4)

> Claude Code 플러그인 — 한국 법령 조회·검색·비교·개정 감지·의무항목 매핑

[국가법령정보센터 공동활용 Open API](https://open.law.go.kr)를 활용하여 한국 법령을 조회하는 Claude Code 플러그인입니다. **DB 없이 curl만으로 동작**하며, PostgreSQL + Prisma 프로젝트에서는 의무항목 매핑 기능이 추가됩니다.

---

## 설치

### 방법 1: 플러그인으로 설치 (권장)

```bash
git clone https://github.com/iamupd/claude-law-skill.git
claude --plugin-dir ./claude-law-skill
```

### 방법 2: 스킬 파일만 복사

```bash
mkdir -p .claude/skills/law
curl -o .claude/skills/law/SKILL.md \
  https://raw.githubusercontent.com/iamupd/claude-law-skill/main/skills/law/SKILL.md
```

---

## 사전 설정

국가법령정보센터 Open API 인증키(OC)가 필요합니다.

1. [open.law.go.kr](https://open.law.go.kr) 회원가입
2. 가입한 이메일 ID가 OC 값입니다

```bash
# 환경변수 설정
export LAW_API_OC_KEY="your@email.com"

# 또는 .env 파일에 추가
echo 'LAW_API_OC_KEY="your@email.com"' >> .env
```

---

## 사용법

```
/law search <키워드>              법령 검색
/law article <법령명> [조항]       조문 조회
/law diff <법령명> <조항> [비교]   API vs 로컬 비교
/law recent [법령명] [일수]        최근 개정 조회
/law history <법령명>             법령 연혁
/law map <O-코드>                의무항목 매핑 (DB 모드)
```

### 예시

```
/law search 안전보건교육
/law article 중대재해처벌법 제4조
/law article 산업안전보건법              # 조항 생략 → 전체 목차
/law diff 산업안전보건법 제29조 ./docs/old-version.txt
/law recent 중대재해처벌법 90
/law history 산업안전보건법
/law map O-01                          # DB 모드 전용
```

---

## 출력 예시

### 조문 조회

```
━━ 중대재해 처벌 등에 관한 법률 제4조 ━━
사업주와 경영책임자등의 안전 및 보건 확보의무

사업주 또는 경영책임자등은 사업주나 법인 또는 기관이 실질적으로
지배·운영·관리하는 사업 또는 사업장에서 종사자의 안전·보건상 ...

  ① 재해예방에 필요한 인력 및 예산 등 안전보건관리체계의 구축 ...
  ② 재해 발생 시 재발방지 대책의 수립 및 그 이행에 관한 조치
  ③ ...

시행일: 2022-01-27 | 법령구분: 법률
소관: 고용노동부
```

### 조문 조회 (DB 모드 — 의무항목 매핑 포함)

```
━━ 중대재해 처벌 등에 관한 법률 제4조 ━━
사업주와 경영책임자등의 안전 및 보건 확보의무

  ① 재해예방에 필요한 인력 및 예산 등 ...
  ② ...

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

### 법령 검색

```
검색 결과: "안전보건교육" (총 5건)

 # │ 법령명                        │ 구분    │ 제개정   │ 시행일
───┼──────────────────────────────┼────────┼─────────┼───────────
 1 │ 산업안전보건법                  │ 법률    │ 일부개정 │ 2024-01-01
 2 │ 산업안전보건법 시행령            │ 대통령령 │ 일부개정 │ 2024-03-01
```

### 법령 연혁

```
━━ 중대재해 처벌 등에 관한 법률 연혁 ━━

공포일       │ 시행일       │ 구분
─────────────┼─────────────┼──────────
2021-01-26   │ 2022-01-27   │ 제정
2023-05-16   │ 2024-01-27   │ 일부개정

현행: 시행 2024-01-27 (2023-05-16 일부개정)
```

---

## 동작 모드

이 스킬은 환경에 따라 두 가지 모드로 동작합니다:

| 모드 | 조건 | 기능 |
|------|------|------|
| **범용 모드** (기본) | 항상 | `search`, `article`, `diff`, `recent`, `history` |
| **DB 모드** (자동 감지) | `prisma/schema.prisma`에 `Regulation` 모델 존재 시 | 위 전체 + `map`, 의무항목 매핑 |

DB 모드는 자동 감지됩니다. Prisma 기반 프로젝트가 아니면 범용 모드로 동작합니다.

---

## 기능 상세

| 커맨드 | 설명 | 모드 |
|--------|------|:----:|
| `/law search <키워드>` | 키워드로 법령 검색 | 범용 |
| `/law article <법령명> [조항]` | 조문 조회 (계층 구조 포함) | 범용 |
| `/law diff <법령명> <조항> [비교대상]` | API 최신 vs 로컬 파일/DB 비교 | 범용 |
| `/law recent [법령명] [일수]` | 최근 N일 개정 법령 조회 | 범용 |
| `/law history <법령명>` | 법령 제정·개정 연혁 | 범용 |
| `/law map <O-코드>` | 의무항목 → 근거법령 역추적 | DB |

---

## 구조

```
claude-law-skill/
├── .claude-plugin/
│   └── plugin.json              # 플러그인 매니페스트
├── skills/
│   └── law/
│       └── SKILL.md             # 스킬 정의 (핵심 파일)
├── README.md                    # 이 문서
└── LICENSE                      # MIT
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
| API 안내 | https://open.law.go.kr |

---

## 지원 플랫폼

| 플랫폼 | 지원 |
|--------|:----:|
| Claude Code CLI | O |
| Claude Code Desktop (Mac/Windows) | O |
| Claude Code Web (claude.ai/code) | O |
| Claude Code IDE Extensions (VS Code, JetBrains) | O |

---

## 관련 프로젝트

- [hwp-extractor](https://github.com/iamupd/hwp-extractor) — HWP/HWPX 문서 텍스트 추출 스킬

---

## 라이선스

![License](https://img.shields.io/badge/License-MIT-green)

MIT
