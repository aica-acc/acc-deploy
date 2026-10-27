# ACC 시작 가이드

처음 받은 사람이 **위에서부터 순서대로** 따라 하면 되는 문서입니다. 기술적인 설명은 [README](../README.md)에 있습니다.

## 0. 내 역할 확인

| 역할 | 할 일 |
|---|---|
| **실행만** (기획·디자인·테스트) | 1 → 2 → 3 → 4 → 5 |
| **개발자** (코드 수정) | 위 + [6. 개발자 추가 작업](#6-개발자-추가-작업) |
| **서버 운영** (팀장) | 위 + [7. 서버 배포 / 도메인](#7-서버-배포--도메인) |

## 1. 설치

| 프로그램 | 비고 |
|---|---|
| [Docker Desktop](https://www.docker.com/products/docker-desktop/) | Windows 는 설치 중 **WSL 2** 사용 선택. 설치 후 Docker Desktop 을 실행해 둔다 |
| [Git](https://git-scm.com/downloads) | |

PC 사양: 디스크 여유 **40GB 이상** (AI 이미지가 약 16GB), 메모리 16GB 권장.

## 2. 받기

```bash
git clone https://github.com/aica-acc/acc-deploy.git
cd acc-deploy
```

**Drive** 의 ACC 원본 자료에서 아래 두 폴더를 받아 `acc-deploy` 안에 그대로 넣는다. (git 에 올릴 수 없는 폰트·이미지·샘플)

```
acc-deploy/
├─ assets/   ← Drive 에서 받음 (ai-data, ai-fonts, samples)
├─ data/     ← Drive 에서 받음 (front-public)
├─ docker-compose.yml
└─ ...
```

## 3. API 키 준비

키는 **두 종류**로 나눠 쓴다.

| 어디서 | 어떤 키 |
|---|---|
| **내 PC (로컬 테스트·개발)** | **각자 개인 발급** 키 |
| **팀 서버** | **팀 공용** 키. 팀장이 관리하며 서버에만 넣는다 |

> 팀 공용 키는 단톡방·Drive·GitHub 에 절대 올리지 않는다. 필요하면 팀장에게 1:1 로 요청한다.
> 개인 키도 마찬가지로 `env/ai.env` 에만 넣고 공유하지 않는다 (`.gitignore` 대상).

### 필수 키 (없으면 AI 서버가 뜨지 않음)

| 키 | 발급 | 비고 |
|---|---|---|
| `OPENAI_API_KEY` | [platform.openai.com/api-keys](https://platform.openai.com/api-keys) | 결제 수단 등록 + 크레딧 충전 필요 |
| `GEMINI_API_KEY` | [aistudio.google.com/apikey](https://aistudio.google.com/apikey) | 영상 생성(Veo)은 결제 설정된 프로젝트 필요 |
| `REPLICATE_API_TOKEN` | [replicate.com/account/api-tokens](https://replicate.com/account/api-tokens) | 결제 수단 등록 필요 |

### 기능별 키 (없으면 해당 기능만 실패)

| 키 | 발급 | 쓰는 기능 |
|---|---|---|
| `CLIPDROP_API_KEY` | [clipdrop.co/apis](https://clipdrop.co/apis) | 에디터 배경 제거 |
| `NAVER_CLIENT_ID`, `NAVER_CLIENT_SECRET` | [developers.naver.com](https://developers.naver.com/apps/) → 애플리케이션 등록 → **데이터랩(검색어트렌드)** 선택 | 키워드·지역 트렌드 |
| `YOUTUBE_API_KEY` | [Google Cloud Console](https://console.cloud.google.com/apis/library/youtube.googleapis.com) → YouTube Data API v3 사용 → 사용자 인증 정보 | 유튜브 트렌드 |
| `SERPAPI_API_KEY` | [serpapi.com](https://serpapi.com/manage-api-key) | 포스터 수집 |
| `CLOUDCONVERT_API_KEY` | [cloudconvert.com](https://cloudconvert.com/dashboard/api/v2/keys) | HWP 기획서 변환 |
| `TAVILY_API_KEY`, `PINECONE_API_KEY`, `HUGGING_FACE_TOKEN` | 각 사이트 | 검색·벡터 검색·모델 다운로드 |

처음에는 **필수 3개만** 넣고 띄워 본 뒤, 필요한 기능의 키를 추가하면 된다.

## 4. 설정 파일 만들기

```bash
cp .env.example .env
cp env/ai.env.example env/ai.env
```

### `env/ai.env` (API 키)

3 에서 준비한 키를 `=` 뒤에 붙여 넣는다. 따옴표·공백 없이.

```
OPENAI_API_KEY=sk-...
GEMINI_API_KEY=AIza...
REPLICATE_API_TOKEN=r8_...
```

경로 항목(`PROJECT_ROOT` 등)은 비워 둔다. 도커가 알아서 채운다.

### `.env` (DB·로그인 설정)

로컬 테스트는 아래 4 줄만 바꾸면 된다. 나머지는 기본값 그대로.

| 항목 | 넣을 값 |
|---|---|
| `MYSQL_ROOT_PASSWORD` | 아무 긴 문자열 (내 PC 전용) |
| `MYSQL_PASSWORD` | 아무 긴 문자열 (위와 다르게) |
| `AUTH_JWT_SECRET` | 32자 이상 아무 문자열 |
| `AUTH_ADMIN_PASSWORD` | 관리자 로그인 비밀번호 (8자 이상) |

> 임의 문자열 만들기: `python -c "import secrets; print(secrets.token_urlsafe(32))"`

## 5. 실행 / 확인

```bash
docker compose pull      # 이미지 받기 (처음엔 오래 걸림)
docker compose up -d     # 실행
```

1. 약 **2분** 기다린다 (AI 서버가 모델을 불러오는 시간).
2. 브라우저에서 http://localhost:5173 접속.
3. **test 계정으로 로그인하기** → 기획서 업로드 화면에 샘플이 자동 입력되면 성공.
4. 관리자: `admin@acc.local` / `.env` 에 넣은 `AUTH_ADMIN_PASSWORD` → `/admin` 접속.

끄기: `docker compose down` (DB 데이터는 남는다)

### 안 될 때

| 증상 | 확인 |
|---|---|
| 화면은 뜨는데 분석·생성이 502 | AI 서버 기동 중. 2분 더 기다린다 |
| 계속 502 | `docker compose logs ai --tail 50` → `... 가 .env에 설정되어 있지 않습니다` 면 필수 키 누락 |
| AI 서버가 계속 재시작 | 위와 같음. 또는 `assets/ai-data/total_trend_images` 폴더 누락 (2 단계 확인) |
| `docker compose` 명령이 안 됨 | Docker Desktop 이 실행 중인지 확인 |
| 로그인이 자꾸 풀림 | `.env` 의 `AUTH_JWT_SECRET` 이 비어 있음 |
| 포트 5173 사용 중 | `.env` 의 `FRONT_PORT` 를 다른 번호로 |

## 6. 개발자 추가 작업

서비스 레포 3개를 `acc-deploy` **안에** clone 한다.

```bash
git clone https://github.com/aica-acc/acc-ai.git
git clone https://github.com/aica-acc/acc-backend.git
git clone https://github.com/aica-acc/acc-front.git
```

- 수정한 코드로 띄우기: `docker compose up -d --build`
- 레포 하나만 단독으로 띄워 개발할 때는 각 레포 README 의 "빠른 실행"을 따른다. 이때는 각 레포의 `.env.example` / `application.properties.example` 를 복사해서 쓴다. (도커용 설정과 별개)
- 프론트 빌드에는 Drive 의 영상 파일이 필요하다: `프론트엔드/video.zip` → `acc-front/src/assets/sections/video/`
- 커밋은 **본인 GitHub 계정**으로: `git config user.name` / `user.email` 확인

## 7. 서버 배포 / 도메인

> 서버 위치는 아직 미정. 결정되면 아래 둘 중 하나를 고른다.

공통:

1. 서버에 1~5 단계를 똑같이 진행한다. 단, `env/ai.env` 에는 **팀 공용 키**를 넣는다.
2. `.env` 를 바꾼다.
   ```
   PUBLIC_URL=https://acc.donghwan.cloud
   AUTH_TEST_LOGIN_ENABLED=false     # 외부 공개 시 test 로그인 차단 (시연용이면 true 유지)
   DEBUG_BIND=127.0.0.1               # 백엔드 직접 포트는 외부에 열지 않는다
   ```
   `MYSQL_*`, `AUTH_JWT_SECRET`, `AUTH_ADMIN_PASSWORD` 는 로컬과 **다른** 값으로 새로 만든다.

도메인은 가비아의 `donghwan.cloud`. 서비스 주소는 서브도메인(예: `acc.donghwan.cloud`)을 쓴다.

### A. 집·노트북에서 서빙 — Cloudflare Tunnel

공인 IP·공유기 포트포워딩이 필요 없다. 대신 **`donghwan.cloud` 전체의 DNS 관리가 Cloudflare 로 옮겨진다** (가비아에서 쓰던 다른 레코드가 있으면 Cloudflare 에 다시 등록).

1. [Cloudflare](https://dash.cloudflare.com/) 가입 → **사이트 추가** → `donghwan.cloud` → Free 플랜.
2. Cloudflare 가 알려주는 **네임서버 2개**를 가비아 → My가비아 → 도메인 → `donghwan.cloud` 관리 → **네임서버 설정** 에 입력. (반영까지 수 분~수 시간)
3. Cloudflare → **Zero Trust → Networks → Tunnels → Create a tunnel** (Cloudflared) → 이름 `acc`.
4. 설치 안내 화면의 토큰(`eyJ...`)을 `.env` 의 `CLOUDFLARE_TUNNEL_TOKEN` 에 넣는다. (설치 명령은 실행하지 않는다 — 도커가 대신 띄운다)
5. **Public Hostname** 추가: 서브도메인 `acc`, 도메인 `donghwan.cloud`, Service `HTTP` / `front:80`.
6. 실행: `docker compose --profile tunnel up -d`

### B. 클라우드 VM(GCP 등)에서 서빙 — Caddy (자동 HTTPS)

DNS 는 가비아에 그대로 둔다.

1. VM 에 **고정 외부 IP** 를 할당하고, 방화벽에서 **80, 443** 포트를 연다.
2. 가비아 → 도메인 → `donghwan.cloud` 관리 → **DNS 설정** → 레코드 추가: 타입 `A`, 호스트 `acc`, 값 `VM 외부 IP`.
3. `.env` 에 `DOMAIN=acc.donghwan.cloud`.
4. 실행: `docker compose --profile caddy up -d` (인증서는 Let's Encrypt 로 자동 발급)

확인: https://acc.donghwan.cloud 접속.
