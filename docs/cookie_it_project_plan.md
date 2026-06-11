# cookie IT
### IT 뉴스 자동 크롤링 · 요약 · Notion 저장 서비스
> 포트폴리오 & 학습용 프로젝트 기획서 · 2026.06.11

---

## 1. 프로젝트 개요

| 항목 | 내용 |
|------|------|
| 프로젝트명 | Daily IT Digest |
| 목적 | 매일 아침 IT 키워드 뉴스를 자동 수집·요약하여 Notion에 정리 |
| 타깃 | 개발자, IT 종사자, 기술 트렌드를 꾸준히 팔로우하고 싶은 사람 |
| 핵심 가치 | 매일 5분 안에 어제의 IT 흐름 파악 |
| 개발 목적 | 포트폴리오 + Python / Docker / PostgreSQL / API 학습 |

---

## 2. 핵심 기능

### ① 키워드 설정
- 사용자가 원하는 키워드 등록 (예: `AI`, `보안`, `LLM`, `스타트업`)
- `config.py` 파일에서 키워드 목록 관리

### ② 뉴스 크롤링
- 국내: ZDNet Korea, IT조선, 디지털타임스 (RSS + HTML 파싱)
- 해외: TechCrunch (RSS), Hacker News (공식 JSON API)
- 전날(어제) 기사만 날짜 필터링

### ③ AI 3줄 요약
- Ollama 로컬 LLM (무료) 기본 사용
- Claude API 교체 가능한 추상화 구조로 설계
- 요약 결과는 DB에 저장 후 Notion 업로드

### ④ PostgreSQL + Docker
- Docker Compose로 PostgreSQL 컨테이너 관리
- 중복 기사 방지, 업로드 상태 추적, 재시도 지원

### ⑤ Notion 자동 저장
- Notion API로 날짜별 페이지 자동 생성
- 키워드별 섹션 분류 후 카드 형태로 정리

### ⑥ 스케줄러
- 매일 오전 8시 APScheduler로 자동 실행
- 실패 시 로그 저장 및 재시도 로직

---

## 3. 시스템 아키텍처

```
[스케줄러: APScheduler]  →  매일 08:00 자동 실행
        ↓
[크롤러 모듈]
  ├── RSS 파싱 (feedparser)          ← TechCrunch, IT조선
  ├── HTML 크롤링 (BeautifulSoup)    ← ZDNet 등
  └── Hacker News API (JSON)        ← 해외 개발자 커뮤니티
        ↓
[필터 모듈]  날짜 필터 + 키워드 매칭
        ↓
[PostgreSQL DB]  중복 체크 → 원문 저장
        ↓
[요약 모듈]  Ollama(기본) / Claude API(교체 가능)
        ↓
[Notion 업로드]  날짜별 페이지 자동 생성
```

---

## 4. 기술 스택

| 영역 | 기술 | 선택 이유 |
|------|------|-----------|
| 언어 | Python | 크롤링·자동화·AI 생태계 최강 |
| 크롤링 | requests + BeautifulSoup4 | 가장 기본적, 학습에 최적 |
| RSS 파싱 | feedparser | 한 줄로 RSS 처리 가능 |
| AI 요약 | Ollama (llama3.2) | 완전 무료, 로컬 실행 |
| AI 요약 (대체) | Claude API (Haiku) | 월 ~2,000원, 나중에 교체 가능 |
| DB | PostgreSQL 15 | 실무 표준, 포트폴리오 어필 |
| 컨테이너 | Docker + Docker Compose | 환경 통일, 배포 편의성 |
| Python-DB 연결 | psycopg2 → SQLAlchemy | 기초 학습 후 ORM으로 업그레이드 |
| Notion 연동 | notion-client | 공식 Python SDK |
| 스케줄러 | APScheduler | 로컬 환경 cron 대체 |
| 환경변수 | python-dotenv | API 키 보안 관리 |
| 배포 DB | Supabase (무료) | PostgreSQL 100% 호환 |

---

## 5. DB 설계 (PostgreSQL)

### articles 테이블 가제

| 컬럼명 | 타입 | 설명 |
|--------|------|------|
| id | PRIMARY KEY | 자동 증가 PK |
| url | TEXT UNIQUE NOT NULL | 기사 URL (중복 체크 핵심) |
| title | TEXT | 기사 제목 |
| source | TEXT | 출처 (techcrunch, zdnet 등) |
| keyword | TEXT | 매칭된 키워드 |
| content | TEXT | 기사 본문 (요약 전 원문) |
| summary | TEXT | AI 3줄 요약 결과 |
| crawled_at | TIMESTAMP DEFAULT NOW() | 크롤링 시각 (타임존 포함) |
| uploaded | BOOLEAN DEFAULT FALSE | Notion 업로드 완료 여부 |

### 중복 방지 흐름

```
크롤링 후 URL 획득
        ↓
DB에서 URL 조회
  ├── 이미 존재 → skip
  └── 없음 → DB 저장 (uploaded=FALSE)
                ↓
            AI 요약 → summary 업데이트
                ↓
            Notion 업로드
                ↓
            uploaded = TRUE 업데이트
            (실패 시 FALSE 유지 → 재시도 가능)
```

---

## 6. Docker 구성

### Docker란?

> "내 컴퓨터에서 돌아가는 환경을 그대로 박스에 담아서 어디서든 똑같이 실행하는 기술"

| 개념 | 비유 | 역할 |
|------|------|------|
| Image (이미지) | 클래스 / 레시피 | 환경 설계도. Docker Hub에서 공식 이미지 다운로드 |
| Container (컨테이너) | 인스턴스 / 요리 결과 | 이미지를 실제로 실행한 것.독립된 환경에서 동작 |
| Docker Compose | 오케스트라 지휘자 | 여러 컨테이너(DB + 앱)를 한 번에 실행/종료 |
| Volume | 외장 하드 | 컨테이너가 꺼져도 데이터가 사라지지 않도록 영구 저장 |

---

## 7. 개발 로드맵

| 단계 | 내용 | 기간 | 학습 포인트 |
|------|------|------|-------------|
| 1단계 | Docker Desktop 설치 + PostgreSQL 컨테이너 실행 | 1일 | Docker 기초, docker-compose |
| 2단계 | DB 연결 + 테이블 생성 (psycopg2) | 1~2일 | PostgreSQL, SQL 기초 |
| 3단계 | RSS 크롤러 구현 (TechCrunch 1개) | 1~2일 | feedparser, requests |
| 4단계 | 날짜·키워드 필터 + DB 저장 | 1~2일 | 데이터 파이프라인 설계 |
| 5단계 | Ollama 설치 + 3줄 요약 연동 | 1~2일 | 로컬 LLM, 추상화 패턴 |
| 6단계 | Notion API 연동 + 페이지 자동 생성 | 2~3일 | 외부 API 연동 |
| 7단계 | APScheduler 자동 실행 | 1일 | 스케줄러, 백그라운드 실행 |
| 8단계 | 국내 소스 추가 + 예외처리 + README | 2~3일 | BeautifulSoup, 코드 품질 |
| 9단계 | Supabase 배포 + Python 컨테이너화 | 2일 | 클라우드 DB, Dockerfile |

**총 예상 기간: 약 3~4주 (학습 병행 기준)**

---

## 19. 향후 확장 아이디어

- Slack / Discord 웹훅 알림 추가
- 키워드별 트렌드 분석 대시보드 (Streamlit)
- SQLAlchemy ORM으로 DB 레이어 업그레이드
- FastAPI로 키워드 관리 웹 인터페이스 제작
- GitHub Actions로 서버 없이 매일 자동 실행

---

*Cookie IT · 기획서 v1.0*
