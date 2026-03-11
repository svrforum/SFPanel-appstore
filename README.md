# SFPanel App Store

SFPanel 앱스토어 레포지토리. 이 저장소의 앱 정의를 기반으로 SFPanel에서 원클릭 설치가 가능합니다.

## 저장소 구조

```
├── categories.json              # 카테고리 목록
└── apps/
    └── {app-id}/
        ├── metadata.json        # 앱 메타데이터 (필수)
        ├── docker-compose.yml   # Docker Compose 파일 (필수)
        └── icon.svg             # 앱 아이콘 (선택 — metadata.json에 icon URL 지정 가능)
```

## 새 앱 추가 방법

### 1. 앱 디렉토리 생성

`apps/` 아래에 앱 ID로 디렉토리를 생성합니다.

**앱 ID 규칙:** 소문자 + 숫자 + 하이픈(`-`), 최대 50자. 정규식: `^[a-z0-9][a-z0-9_-]{0,49}$`

```bash
mkdir apps/my-app
```

### 2. metadata.json 작성

```jsonc
{
  "id": "my-app",                          // 앱 ID (디렉토리명과 동일)
  "name": "My App",                        // 표시 이름
  "description": {                         // 다국어 설명 (ko, en 필수)
    "ko": "한글 설명",
    "en": "English description"
  },
  "category": "dev",                       // 카테고리 ID (아래 목록 참조)
  "version": "1.0.0",                      // 앱스토어 패키지 버전 (실제 앱 버전은 GitHub Releases에서 자동 가져옴)
  "website": "https://example.com",        // 공식 웹사이트
  "source": "https://github.com/org/repo", // GitHub 소스 저장소 (버전 자동 감지 + README 표시에 사용)
  "icon": "https://...svg or png",         // (선택) 아이콘 URL. 미지정 시 apps/{id}/icon.svg 사용
  "ports": [8080],                         // 사용하는 포트 목록
  "features": [                            // (선택) 주요 기능 카드 (최대 4개 권장)
    {
      "title": { "ko": "기능명", "en": "Feature" },
      "description": { "ko": "설명", "en": "Description" },
      "icon": "🚀"                         // 이모지 아이콘
    }
  ],
  "env": [                                 // .env 파일에 들어갈 환경변수 정의
    {
      "key": "PORT",                       // 환경변수 키
      "label": { "ko": "외부 포트", "en": "External Port" },
      "type": "port",                      // 입력 타입 (아래 참조)
      "default": "8080"                    // 기본값
    },
    {
      "key": "DB_PASSWORD",
      "label": { "ko": "DB 비밀번호", "en": "DB Password" },
      "type": "password",
      "required": true,                    // 필수 여부
      "generate": true                     // true면 설치 시 32자 랜덤 생성
    },
    {
      "key": "STORAGE_TYPE",
      "label": { "ko": "저장소 타입", "en": "Storage Type" },
      "type": "select",
      "default": "local",
      "options": ["local", "s3"]           // select 타입일 때 선택지
    }
  ]
}
```

### 3. docker-compose.yml 작성

환경변수는 `${VAR_NAME:-default}` 형태로 참조합니다. 설치 시 metadata.json의 `env` 정의에 따라 `.env` 파일이 자동 생성됩니다.

```yaml
services:
  my-app:
    image: org/my-app:latest
    container_name: my-app
    restart: unless-stopped
    ports:
      - "${PORT:-8080}:80"
    environment:
      - DB_PASSWORD=${DB_PASSWORD}
    volumes:
      - my-app-data:/data

volumes:
  my-app-data:
```

**규칙:**
- `container_name`은 앱 ID와 동일하게 설정
- 볼륨은 named volume 사용 (호스트 바인드 마운트 지양)
- `restart: unless-stopped` 권장
- 포트는 `${PORT:-기본값}:컨테이너포트` 형태로 사용자가 변경 가능하게

### 4. (선택) 아이콘 추가

- `apps/{id}/icon.svg` 파일로 추가하거나
- `metadata.json`의 `icon` 필드에 외부 URL 지정

SVG 권장. 48x48 ~ 128x128 크기에 최적화.

## 카테고리 목록

| ID | 한글 | English |
|----|------|---------|
| `media` | 미디어 | Media |
| `cloud` | 클라우드 | Cloud |
| `security` | 보안 | Security |
| `monitoring` | 모니터링 | Monitoring |
| `dev` | 개발 | Development |

새 카테고리가 필요하면 `categories.json`에 추가합니다:

```json
{
  "id": "new-category",
  "name": { "ko": "새 카테고리", "en": "New Category" },
  "icon": "Folder"
}
```

## env 타입 설명

| type | 설명 | UI 동작 |
|------|------|---------|
| `text` | 일반 텍스트 | 텍스트 입력 |
| `port` | 포트 번호 | 숫자 입력 |
| `password` | 비밀번호 | 마스킹 입력 + 눈 토글 |
| `select` | 선택형 | 드롭다운 (`options` 필드 필요) |

## 동작 방식

SFPanel이 이 저장소를 사용하는 흐름:

1. **캐시 로드** — GitHub API로 `apps/` 디렉토리 목록 + 각 `metadata.json` 가져옴 (1시간 캐시)
2. **버전 감지** — `source` 필드의 GitHub 저장소에서 최신 릴리즈 태그를 자동으로 가져와 표시
3. **README 표시** — `source` 필드의 GitHub 저장소 README.md를 가져와 상세 페이지에 표시 (main → master → develop 순으로 시도)
4. **설치** — `docker-compose.yml` 다운로드 → `/opt/stacks/{app-id}/` 에 저장 → `env` 정의에 따라 `.env` 생성 → `docker compose up -d` 실행

## 예시: 기존 앱 참조

새 앱 추가 시 기존 앱을 참조하세요:

- **단순 (환경변수 적음)**: `apps/uptime-kuma/`
- **비밀번호 자동 생성**: `apps/vaultwarden/` (`generate: true`)
- **다중 서비스 + DB**: `apps/nextcloud/`, `apps/immich/`
- **포트 여러 개**: `apps/nginx-proxy-manager/`
