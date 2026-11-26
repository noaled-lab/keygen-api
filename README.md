# Keygen API (self-hosted fork)

[keygen-sh/keygen-api](https://github.com/keygen-sh/keygen-api)를 Docker Compose로 셀프 호스팅하기 위한 포크입니다.
원본 README는 [README.old.md](README.old.md)에 보관되어 있습니다.

아래에서 `<...>`로 표시된 값은 각 환경에 맞게 바꿔서 사용합니다.

## 관련 파일

| 파일 | 설명 |
| --- | --- |
| [`docker-compose.noaled.yaml`](docker-compose.noaled.yaml) | 운영 compose (api, worker, postgres, redis, portal + setup/console/migrate 프로필) |
| [`noaled.env.example`](noaled.env.example) | 서버 `.env` 템플릿 |
| [`backup.env.example`](backup.env.example) | 서버 `.env.backup` 템플릿 (DB 백업: S3 · 암호화 · 일정 · 알림) |
| [`.github/workflows/deploy-noaled.yml`](.github/workflows/deploy-noaled.yml) | 배포 워크플로 (수동 실행) |
| [`.github/workflows/test.yml`](.github/workflows/test.yml) | 원본 CI (push/PR 시 자동 실행) |

## 구성

```
인터넷 ─▶ 리버스 프록시 (외부 네트워크 app)
            ├─ <portal 도메인>  ─▶ keygen-portal (80)
            └─ <api 도메인>     ─▶ keygen-api (PORT, 기본 3000)
                                     ├─ keygen-postgres ── WAL · 백업 ─▶ RustFS (S3)
                                     └─ keygen-redis
                                  keygen-worker (sidekiq)
                                  keygen-backup (예약 백업, 아래 "DB 백업")
```

- 리버스 프록시는 이 저장소 밖에서 관리하며, 외부 Docker 네트워크 `app`을 미리 만들어 둬야 합니다.
- `keygen-api`, `keygen-portal`만 `app` 네트워크에 붙습니다. 컨테이너 이름은 `<project>-keygen-api-1` 형태가 됩니다.
- portal 이미지는 비공개 GHCR 이미지이고, 나머지는 Docker Hub 공개 이미지입니다.
- portal은 빌드 시점에 API 호스트, edition/mode, account ID가 고정됩니다. 서버 `.env`의 값과 일치해야 합니다.

## 배포

GitHub Actions → **Deploy with Docker Compose** → **Run workflow** (수동 실행만 가능).

워크플로가 하는 일:

1. `docker-compose.noaled.yaml`을 서버 `/home/<REMOTE_USER>/keygen`으로 복사
2. `DOT_ENV`·`DOT_ENV_BACKUP` 시크릿을 base64 디코드해서 `.env`·`.env.backup` 생성 (권한 600)
3. `GITHUB_TOKEN`으로 `ghcr.io` 로그인
4. `pull --ignore-buildable` → `build`(keygen-postgres: postgres + WAL-G) → `run --rm keygen-migrate` → `up -d --remove-orphans` → `image prune`

### 필요한 Repository Secrets

| Secret | 설명 |
| --- | --- |
| `REMOTE_HOST` | 서버 주소 |
| `REMOTE_USER` | SSH 사용자 (배포 경로가 `/home/<REMOTE_USER>/keygen`이 됨) |
| `REMOTE_SSH_KEY` | SSH 개인키 |
| `DOT_ENV` | 서버 `.env` 전체를 base64 인코딩한 값 (`base64 -w0 .env`) |
| `DOT_ENV_BACKUP` | 서버 `.env.backup` 전체를 base64 인코딩한 값 (`base64 -w0 .env.backup`). 없으면 배포가 멈춤 ([DB 백업](#db-백업)) |

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

## DB 백업

[WAL-G 공식 문서](https://github.com/wal-g/wal-g/blob/master/docs/PostgreSQL.md) 방식 그대로 씁니다.

- `keygen-postgres`: 공식 `postgres:17.5`에 WAL-G v3.0.10 실행 파일을 더한 이미지(compose 안의 Dockerfile, 체크섬 고정). `archive_command`로 WAL을 S3 호환 저장소(RustFS)에 올립니다.
- `keygen-backup`: 매일 `BACKUP_TIME`에 `backup-push`. 증분이 `WALG_DELTA_MAX_STEPS`번 쌓이면 다음은 자동으로 전체입니다. 끝나면 `delete retain FULL`로 정리하고 웹훅으로 알립니다.
- 알림(웹훅): 백업 성공·실패, WAL 업로드 실패(`pg_stat_archiver`, 5분마다 확인), `BACKUP_MAX_AGE_HOURS`(기본 26시간) 넘게 성공한 백업 없음. 문제가 풀리면 한 번 더 알립니다.
- 설정은 전부 서버의 `.env.backup`입니다. [`backup.env.example`](backup.env.example)을 복사해 채웁니다. 다른 서버에서는 이 파일만 바꿉니다(`WALG_S3_PREFIX`는 서버마다 다르게).

### 처음 적용 (운영 중인 DB에도 같음)

데이터 볼륨과 Postgres 버전은 그대로이고, 설정 적용을 위해 Postgres가 한 번 재시작됩니다(몇 초).

1. RustFS에 버킷과, 그 경로만 쓸 수 있는 백업 전용 키를 만듭니다. 버킷 수명 주기 규칙을 쓴다면 `backup_reservations/`는 빼 둡니다.
2. `.env.backup`을 채웁니다. `WALG_LIBSODIUM_KEY`는 `openssl rand -hex 32`로 만들고 **비밀번호 관리자에도 보관**합니다. Keygen `.env`의 `ENCRYPTION_*`·`SECRET_KEY_BASE`도 함께 보관합니다. 이 값들이 없으면 백업이 있어도 복원할 수 없습니다.
3. `DOT_ENV_BACKUP` 시크릿을 등록합니다(`base64 -w0 .env.backup`).
4. 만약을 위해 서버에서 덤프를 받아 둡니다: `docker compose -f docker-compose.noaled.yaml exec -T keygen-postgres pg_dump -U <POSTGRES_USER> -Fc <POSTGRES_DB> > before-walg.dump`
5. 배포 워크플로를 실행합니다. `keygen-backup` 로그에 `DB 백업 성공`이 나오면 끝입니다.

### 명령

```bash
C="docker compose -f docker-compose.noaled.yaml"
$C logs --tail=50 keygen-backup
$C exec keygen-backup wal-g backup-list --detail
$C exec keygen-backup wal-g backup-push /var/lib/postgresql/data --full --wait-for-archive   # 지금 전체 백업
$C exec keygen-backup wal-g wal-verify integrity timeline   # 저장소의 WAL이 끊김 없이 있는지
$C exec keygen-postgres psql -U <POSTGRES_USER> -c "select * from pg_stat_archiver"   # WAL 업로드 상태
```

### 복원

빈 볼륨에 마지막 백업을 받고(`backup-fetch`), `recovery.signal`과 `restore_command`를 넣고 Postgres를 시작하면 저장소의 WAL을 끝까지 적용한 뒤 열립니다.
마지막으로 올라간 WAL까지만 복구됩니다(장애 직전 최대 5분은 잃을 수 있음). 볼륨 이름 `keygen_postgres`는 배포 폴더 이름(`keygen`)을 따릅니다.

```bash
C="docker compose -f docker-compose.noaled.yaml"
$C stop keygen-api keygen-worker keygen-backup keygen-postgres
# 1) 기존 볼륨은 지우지 말고 keygen_postgres_old로 복사해 보관한 뒤 비운다
docker run --rm -v keygen_postgres:/from -v keygen_postgres_old:/to postgres:17.5 cp -a /from/. /to/
$C rm -f keygen-postgres keygen-backup && docker volume rm keygen_postgres
# 2) 마지막 백업(증분이면 딸린 전체까지)을 받고 복구 설정을 넣는다
$C run --rm -T --no-deps -u postgres --entrypoint bash keygen-postgres -s <<'EOF'
set -e
wal-g backup-fetch "$PGDATA" LATEST
# wal-fetch는 WAL이 없으면 74(복구 끝). 그 밖의 오류(S3 장애 등)는 255로 바꿔 Postgres가 멈추게 한다
cat >> "$PGDATA/postgresql.auto.conf" <<'CONF'
restore_command = 'wal-g wal-fetch %f %p || { [ $? -eq 74 ] && exit 1; exit 255; }'
CONF
# 특정 시각까지만: 위 CONF 안에 recovery_target_time = '2026-11-26 14:30:00+09' 과 recovery_target_action = 'promote' 를 더 넣는다
touch "$PGDATA/recovery.signal"
EOF
# 3) 시작하면 WAL을 적용하고 열린다 (logs keygen-postgres에 "archive recovery complete")
$C up -d
# 4) 복원한 DB의 새 전체 백업을 바로 만든다
$C exec keygen-backup wal-g backup-push /var/lib/postgresql/data --full --wait-for-archive
```

복원 시험은 운영 볼륨 대신 임시 볼륨에 받아서 합니다. 이때 Postgres는 `-c archive_mode=off`로 띄웁니다. 그래야 복원한 DB가 운영 저장소에 WAL을 올리지 않습니다.

**주의**: 경로(`WALG_S3_PREFIX`) 하나에는 DB 하나만 백업합니다. `keygen:setup`으로 DB를 새로 만들었거나, 다른 서버이거나, Postgres 메이저 버전을 올렸다면 새 경로를 씁니다.

## 개발 / 테스트

테스트 실행 방법과 규칙은 [AGENTS.md](AGENTS.md)를 참고하세요. 항상 `bundle exec rake test:*`로 실행합니다.
