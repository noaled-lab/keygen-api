# Keygen API (self-hosted fork)

[keygen-sh/keygen-api](https://github.com/keygen-sh/keygen-api)를 Docker Compose로 셀프 호스팅하기 위한 포크입니다.
원본 README는 [README.old.md](README.old.md)에 보관되어 있습니다.

아래에서 `<...>`로 표시된 값은 각 환경에 맞게 바꿔서 사용합니다.

## 관련 파일

| 파일 | 설명 |
| --- | --- |
| [`docker-compose.noaled.yaml`](docker-compose.noaled.yaml) | 운영 compose (api, worker, postgres, redis, portal + setup/console/migrate 프로필) |
| [`noaled.env.example`](noaled.env.example) | 서버 `.env` 템플릿 |
| [`.github/workflows/deploy-noaled.yml`](.github/workflows/deploy-noaled.yml) | 배포 워크플로 (수동 실행) |
| [`.github/workflows/test.yml`](.github/workflows/test.yml) | 원본 CI (push/PR 시 자동 실행) |

## 구성

```
인터넷 ─▶ 리버스 프록시 (외부 네트워크 app)
            ├─ <portal 도메인>  ─▶ keygen-portal (80)
            └─ <api 도메인>     ─▶ keygen-api (PORT, 기본 3000)
                                     ├─ keygen-postgres
                                     └─ keygen-redis
                                  keygen-worker (sidekiq)
```

- 리버스 프록시는 이 저장소 밖에서 관리하며, 외부 Docker 네트워크 `app`을 미리 만들어 둬야 합니다.
- `keygen-api`, `keygen-portal`만 `app` 네트워크에 붙습니다. 컨테이너 이름은 `<project>-keygen-api-1` 형태가 됩니다.
- portal 이미지는 비공개 GHCR 이미지이고, 나머지는 Docker Hub 공개 이미지입니다.
- portal은 빌드 시점에 API 호스트, edition/mode, account ID가 고정됩니다. 서버 `.env`의 값과 일치해야 합니다.

## 배포

GitHub Actions → **Deploy with Docker Compose** → **Run workflow** (수동 실행만 가능).

워크플로가 하는 일:

1. `docker-compose.noaled.yaml`을 서버 `/home/<REMOTE_USER>/keygen`으로 복사
2. `DOT_ENV` 시크릿을 base64 디코드해서 `.env` 생성 (권한 600)
3. `GITHUB_TOKEN`으로 `ghcr.io` 로그인
4. `pull` → `run --rm keygen-migrate` → `up -d --remove-orphans` → `image prune`

### 필요한 Repository Secrets

| Secret | 설명 |
| --- | --- |
| `REMOTE_HOST` | 서버 주소 |
| `REMOTE_USER` | SSH 사용자 (배포 경로가 `/home/<REMOTE_USER>/keygen`이 됨) |
| `REMOTE_SSH_KEY` | SSH 개인키 |
| `DOT_ENV` | 서버 `.env` 전체를 base64 인코딩한 값 (`base64 -w0 .env`) |

`.env`는 `noaled.env.example`을 복사해서 채웁니다. `KEYGEN_ACCOUNT_ID`는 portal 빌드에 들어간 ID와 같아야 합니다.

## 최초 설치 (DB가 비어 있을 때 1회)

배포 워크플로는 `migrate`만 실행하고 **계정 생성(`keygen:setup`)은 하지 않습니다.**
새 DB에서는 API가 `Account ... does not exist (run rake keygen:setup to create it)`를 출력하며 재시작을 반복하므로,
서버에서 직접 한 번 실행해야 합니다.

```bash
cd /home/<REMOTE_USER>/keygen
docker compose -f docker-compose.noaled.yaml stop keygen-api keygen-worker
docker compose -f docker-compose.noaled.yaml --profile setup run --rm keygen-setup
docker compose -f docker-compose.noaled.yaml up -d
curl -s https://<api 도메인>/v1/ping
```

- `.env`의 `KEYGEN_ACCOUNT_ID`, `KEYGEN_ADMIN_EMAIL`, `KEYGEN_ADMIN_PASSWORD`로 계정을 만듭니다 (없으면 TTY로 물어봄).
- ⚠️ `keygen:setup`은 내부적으로 `db:schema:load`를 실행해 **DB를 초기화**합니다. 데이터가 있는 DB에서는 절대 다시 실행하지 마세요.

## 운영 명령

root로 접속했다면 `~`가 `/root`이므로 반드시 절대 경로로 이동합니다.

```bash
cd /home/<REMOTE_USER>/keygen
docker compose -f docker-compose.noaled.yaml ps -a
docker compose -f docker-compose.noaled.yaml logs --tail=100 keygen-api
docker compose -f docker-compose.noaled.yaml logs --tail=100 keygen-worker
docker compose -f docker-compose.noaled.yaml --profile console run --rm keygen-console   # rails console
docker compose -f docker-compose.noaled.yaml --profile migrate run --rm keygen-migrate   # 마이그레이션
```

## 트러블슈팅

### 배포 시 `keygen-portal Error denied` / `Error response from daemon: denied`

`GITHUB_TOKEN`은 접근 권한이 부여된 패키지만 읽을 수 있습니다. portal 패키지가 다른 저장소에서 올라간 것이면 기본적으로 이 저장소의 토큰으로는 pull이 거부됩니다.
portal 이미지 pull이 실패하면 나머지도 `Interrupted`되고 `set -e`로 배포 전체가 중단됩니다.

- 해결: GitHub 조직의 **Packages → (portal 패키지) → Package settings → Manage Actions access**에 이 저장소를 **Read** 권한으로 추가.
- 대안: `read:packages` 권한의 PAT를 시크릿으로 등록해 `GHCR_TOKEN`으로 사용.
- 참고: `GITHUB_TOKEN`은 워크플로 종료 후 만료되므로, 서버에서 나중에 수동으로 portal 이미지를 pull해야 한다면 PAT 방식이 안정적입니다.

### Portal에서 이메일 입력 후 "Service is unavailable."

portal은 API 호출이 실패하면 이 문구를 보여줍니다. 대부분 API 쪽 문제입니다.

1. `curl -i https://<api 도메인>/v1/ping` → `502 Bad Gateway`면 프록시 뒤의 `keygen-api`가 떠 있지 않은 것.
2. `docker network inspect app`에 keygen-api 컨테이너가 있는지 확인.
3. `ps -a` / `logs keygen-api`로 상태 확인:
   - 컨테이너가 없음 → 배포가 `pull`에서 실패해 `up`까지 못 감 (위 GHCR 항목 참고).
   - `Account ... does not exist`로 재시작 반복 → [최초 설치](#최초-설치-db가-비어-있을-때-1회) 진행.
   - 정상 기동 중 → 프록시의 `proxy_pass` 대상(컨테이너명/`PORT`) 확인.

### `keygen-worker`가 `Restarting`

대부분 API와 같은 원인(계정 미생성, `.env` 누락)입니다. `logs keygen-worker`로 확인하세요.

## 개발 / 테스트

테스트 실행 방법과 규칙은 [AGENTS.md](AGENTS.md)를 참고하세요. 항상 `bundle exec rake test:*`로 실행합니다.
