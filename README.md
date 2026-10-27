# acc-deploy

ACC 배포 설정 레포. `docker compose` 로 db + ai + backend + front 를 한 번에 띄운다. (DB 스키마는 백엔드 기동 시 자동 마이그레이션)

> **처음이라면 [시작 가이드(docs/SETUP.md)](docs/SETUP.md) 부터.** 설치 → 키 발급 → 설정 → 실행 → 서버·도메인 순서로 정리되어 있다.
> 문제 해결 기록은 [docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md), 진행 상태 설계 변경은 [docs/DESIGN-server-progress.md](docs/DESIGN-server-progress.md).

```
브라우저 ──▶ front(nginx) ──/api/, /static/──▶ backend ──▶ db (MySQL)
                                                 └──▶ ai (FastAPI, 외부 비노출)
```

## 받기

| 용도 | 필요한 것 |
|---|---|
| **이미지 받아서 실행** (서버, 테스트) | 이 레포 + `assets/`, `data/` (원본 자료) |
| **소스에서 빌드** (개발, 이미지 갱신) | 위 + 서비스 레포 3개를 이 폴더 안에 clone |

```bash
git clone https://github.com/aica-acc/acc-deploy.git
cd acc-deploy
# 소스에서 빌드할 때만
git clone https://github.com/aica-acc/acc-ai.git
git clone https://github.com/aica-acc/acc-backend.git
git clone https://github.com/aica-acc/acc-front.git
```

서비스 레포(`acc-*/`), 비밀값(`.env`, `env/ai.env`), 원본 자료(`assets/`, `data/`)는 `.gitignore` 대상이라 이 레포에 올라가지 않는다.

### 설정 파일이 두 곳인 이유

- 각 서비스 레포의 `.env.example` / `application.properties.example` → **레포 단독 개발용** (`uvicorn`, `mvnw spring-boot:run`, `npm run dev`)
- 이 레포의 `.env` / `env/ai.env` → **도커 배포용**. 여러 서비스가 같이 쓰는 값(DB 계정, `PUBLIC_URL` 등)을 한 곳에서 관리하고, 컨테이너 주소(`ai:5000`, `db:3306`)는 compose 가 넣어준다. 이미지에는 비밀값이 들어가지 않는다.

## 구성 / 고정 버전 (2025-12 기준, 소스의 lock 파일 그대로)

| 서비스 | 이미지 | 근거 |
|---|---|---|
| db | `mysql:8.4.7` | `acc-backend/db` 의 DDL (JSON 컬럼 사용) |
| ai | `python:3.11.14-slim-bookworm` + uv `0.9.17` | `.python-version` 3.11, `uv.lock` 으로 패키지 버전 고정 |
| backend | 빌드 `maven:3.9.11-eclipse-temurin-17` / 실행 `eclipse-temurin:17.0.17_10-jre-jammy` | `pom.xml` Spring Boot 3.4.10, Java 17 |
| front | 빌드 `node:22.21.1-alpine3.22` / 실행 `nginx:1.28.1-alpine` | `package-lock.json` (Vite 7, React 19) |

## 준비

1. `.env.example` → `.env` 복사 후 값 입력 (접속 주소 `PUBLIC_URL`, 포트, DB 계정, 터널 토큰). 로컬 테스트는 기본값 그대로 두고 DB 비밀번호만 정하면 된다.
2. `env/ai.env.example` → `env/ai.env` 복사 후 API 키 입력 (OpenAI, Replicate, Gemini, ClipDrop, YouTube, Naver 등). **키가 비어 있으면 AI 서버가 시작 중에 종료된다.** 경로 변수(`PROJECT_ROOT` 등)는 compose 가 덮어쓴다.
3. 레포에 없는 파일들 (원본 자료 `ACC 필수 폴더/` 의 `ACC_*_폴더별경로.txt` 기준)

| 원본 자료 위치 | 놓을 위치 | 비고 |
|---|---|---|
| `AI/fonts/Jalnan2TTF.ttf` | `assets/ai-fonts/` | 컨테이너 `app/fonts` |
| `AI/nanum-all_new`, `AI/total_trend_images`, `AI/report` | `assets/ai-data/` | 컨테이너 `app/data` (`total_trend_images` 는 없으면 AI 서버 시작 실패) |
| `AI/layout_templates.zip` (압축 해제) | `assets/ai-data/editor/layout_templates/` | |
| `프론트엔드/poster/*` (few-shot) | `data/front-public/data/promotion/M000001/25/poster/` | 미리 만들어 둬야 한다 |
| `프론트엔드/video.zip` (압축 해제) | `acc-front/src/assets/sections/video/` | `*.mp4` 는 `.gitignore` 대상이라 레포에 없다. 없으면 프론트 빌드 실패 |
| `promotion` 폴더 (`poster_1~8`, `mascot_1~7`) | `acc-front/src/assets/promotion/{poster,mascot}/` | 원본 자료에 포함되어 있지 않았다. 없으면 1x1 임시 이미지로 대체되어 빌드는 되지만 선택 카드 이미지가 비어 보인다 |

4. DB 스키마는 **백엔드가 기동할 때 Flyway 가 자동으로 적용**한다 (`acc-backend/src/main/resources/db/migration/V1~V3`). 컴포즈의 `db` 는 빈 데이터베이스만 만들면 된다.
   - 새 DB: V1~V3 이 순서대로 적용된다.
   - 이미 테이블이 있는 기존 DB(Flyway 이력 없음): 자동으로 baseline 후 V1 부터 적용한다. V1~V3 은 여러 번 실행해도 안전하게 작성되어 있다.
   - 앱 없이 적용만 하려면: `cd acc-backend && ./mvnw flyway:migrate -Dflyway.url="jdbc:mysql://localhost:3306/acc_database?serverTimezone=UTC&characterEncoding=UTF-8" -Dflyway.user=... -Dflyway.password=...`
   - 끄려면 환경변수 `SPRING_FLYWAY_ENABLED=false`. 스키마를 바꿀 때는 기존 파일을 고치지 말고 `V4__설명.sql` 처럼 새 파일을 추가한다.

## 로그인 / 권한 / 테스트 계정

메인 화면에 두 개의 시작 버튼이 있다.

| 버튼 | 동작 |
|---|---|
| **test 계정으로 로그인하기** | 비밀번호 없이 test 계정(`M000001`, 권한 TEST)으로 로그인 → 기획서 업로드 화면에 샘플 기획서(PDF) · 축제명 · 키워드 · 테마가 자동 입력된다. "분석 시작하기"만 눌러 다음 단계로 진행 |
| **실제 이용하기** | 이메일 + 비밀번호 로그인 / 회원가입 (BCrypt + JWT). 가입한 회원은 USER 권한 |

- 권한: `USER`(일반) / `TEST`(샘플 자동 입력) / `ADMIN`(관리자 화면 `/admin`: 회원 목록, 권한 변경, 계정 정지, 회원 데이터 초기화)
- 권한 변경·정지는 **즉시** 적용된다 (백엔드가 매 요청마다 DB 에서 확인).
- 로그인하지 않으면 `/`, `/login` 외의 모든 화면·API 가 막힌다 (`/api/auth/*` 와 이미지 경로 `/static/**`, `/poster-images/**` 만 공개).
- 계정 초기화: 백엔드 기동 시 test 계정은 항상 보장되고, `.env` 의 `AUTH_ADMIN_PASSWORD`(8자 이상)를 주면 관리자 계정이 생성된다.
- 외부에 공개하는 서버는 `.env` 에서 `AUTH_TEST_LOGIN_ENABLED=false`, `AUTH_JWT_SECRET` 설정을 권장한다.
- 마이페이지: **프로젝트**(목록/상세), **저장소**(`/mypage/drive`: 내 모든 홍보물을 프로젝트·종류별로 필터, 미리보기, 다운로드), **마이프로필**(`/mypage/profile`: 이름·연락처 수정, 비밀번호 변경, 내 활동 요약. test 계정은 변경 불가). 없는 주소는 404 페이지로 연결된다.
- 샘플 기획서는 `assets/samples/proposals/` (`samples.json` + 파일). 추가하려면 파일을 넣고 `samples.json` 에 항목을 더하면 된다.
  - **디스크 파일명은 영문**(`<id>.pdf`)으로 둔다. 한글 파일명은 zip 압축·해제 과정에서 깨져 샘플을 찾지 못한다(404). 사용자에게 보이는 한글 이름은 `fileName` 에 적는다.
  - 예: `{"id": "damyang-santa", "title": "제7회 담양산타축제", "file": "damyang-santa.pdf", "fileName": "제7회 담양산타축제.pdf", "theme": "...", "keywords": [...]}`

## 원본 자료 공유 (Drive)

코드와 설정은 git 으로 받고, git 에 올릴 수 없는 것만 Drive 로 공유한다.

- 공유: `assets/`, `data/`, `acc-front/src/assets/sections/video/*.mp4`
- 절대 공유하지 않음: `.env`, `env/ai.env` (DB 비밀번호, JWT 키, API 키)

## 실행

이미지는 도커 허브(`scv3114/acc-ai`, `scv3114/acc-backend`, `scv3114/acc-front`)에 올려져 있다. 버전은 `.env` 의 `ACC_TAG` (날짜 태그, `latest` 도 같은 이미지). 소스 없이 받아서 쓰려면:

```
docker compose pull
docker compose up -d
```

소스에서 직접 빌드하려면 아래처럼 한다 (AI 이미지는 첫 빌드가 오래 걸린다).

```
docker compose up -d --build
```

이미지를 도커 허브에 올리려면 (`docker login -u scv3114` 필요):

```bash
# .env 의 ACC_TAG 를 새 날짜로 바꾼 뒤
ACC_TAG=$(grep '^ACC_TAG=' .env | cut -d= -f2)
docker compose build ai backend front
docker compose push ai backend front
for s in ai backend front; do
  docker tag scv3114/acc-$s:$ACC_TAG scv3114/acc-$s:latest && docker push scv3114/acc-$s:latest
done
```

- 접속: http://localhost:5173 (nginx 한 곳으로 `/`=프론트, `/api/`·`/static/`=백엔드)
- **백엔드가 모든 데이터의 중심이다.** 브라우저 → nginx → 백엔드 → AI·DB. AI 정적 파일(`/static/**`)도 백엔드가 중계한다.
- AI 서버와 DB 는 호스트에 포트를 열지 않는다. 백엔드만 내부 네트워크(`ai:5000`, `db:3306`)로 접근한다.
- 디버깅용 직접 포트: 백엔드 http://localhost:8081 (기본은 127.0.0.1 에만 바인딩). AI 를 직접 보려면 `docker compose exec ai curl localhost:5000/`.
- AI 서버는 모델 라이브러리 로딩으로 기동에 약 2분 걸린다. 그동안 AI 를 쓰는 API 와 `/static/` 은 502 가 나온다.
- 외부 공개: `docker compose --profile tunnel up -d` (Cloudflare Tunnel) 또는 `--profile caddy` (GCP 등 공인 IP, 자동 HTTPS). 설정은 `.env` 참고.

## 알려진 사항

- `acc-ai` 는 `uv.lock` 대로 설치하므로 torch/CUDA 계열 때문에 이미지가 매우 크다 (수 GB).
- 영상 생성은 `ffmpeg` 를 사용하며 AI 이미지에 apt 로 설치되어 있다 (Windows 의 PATH 설정 불필요).
