# Keygen DB 백업 (WAL-G → S3 호환 저장소 / RustFS)

Keygen의 Postgres를 WAL-G로 S3 호환 저장소(RustFS)에 백업합니다.

- **WAL 연속 보관**: 바뀐 내용이 늦어도 5분 안에 올라갑니다(`archive_timeout=300`).
- **예약 백업**: 기본 매일 03:00. 일요일은 전체, 나머지 날은 증분입니다.
- **보관**: 전체 백업 4개(약 4주)와, 거기 딸린 증분·WAL을 남깁니다.
- **암호화**: 올리기 전에 WAL-G가 암호화합니다(libsodium). 저장소에는 암호문만 남습니다.
- **알림**: 성공·실패 웹훅, 그리고 감시 서비스(Healthchecks / Uptime Kuma Push)

설정은 전부 서버의 `.env.backup`(BACKUP_*)으로 넣습니다. compose 파일과 이미지에는 버킷·주소·키가 없습니다.

## 구성

| 서비스 | 이미지 | 하는 일 |
| --- | --- | --- |
| `keygen-postgres` | `ghcr.io/noaled-lab/keygen-postgres:17.5-walg3.0.9` | 공식 `postgres:17.5` + WAL-G. `archive_command='walg wal-push %p'`로 WAL을 올림 |
| `keygen-backup` | 같은 이미지 | `BACKUP_SCHEDULE`마다 `keygen-backup` 실행. 데이터 볼륨은 **읽기 전용**으로만 붙음 |

| 파일 | 설명 |
| --- | --- |
| [`backup/postgres/Dockerfile`](backup/postgres/Dockerfile) | 이미지. WAL-G v3.0.9, supercronic v0.2.51 (체크섬 고정) |
| [`backup/postgres/bin/walg`](backup/postgres/bin/walg) | `BACKUP_*` → WAL-G·S3 환경변수로 옮긴 뒤 `wal-g`를 실행하는 래퍼 |
| [`backup/postgres/bin/keygen-backup`](backup/postgres/bin/keygen-backup) | 백업 1회: 백업 → 보관 정리 → WAL 검증 → WAL 보관 상태 확인 → 알림 |
| [`backup/postgres/bin/keygen-backup-schedule`](backup/postgres/bin/keygen-backup-schedule) | `keygen-backup` 컨테이너 시작 명령. 이 DB의 백업이 없으면 바로 전체 백업 |
| [`backup/postgres/bin/keygen-restore`](backup/postgres/bin/keygen-restore) | 빈 데이터 폴더에 백업을 받아 복구를 준비 |
| [`backup.env.example`](backup.env.example) | `.env.backup` 템플릿 |
| [`.github/workflows/build-postgres.yml`](.github/workflows/build-postgres.yml) | 이미지 빌드·GHCR 업로드 (amd64·arm64) |

## 처음 적용 (이미 운영 중인 DB에도 같음)

데이터 볼륨(`postgres`)과 Postgres 버전(17.5)은 그대로라 데이터 이전은 없습니다.
`archive_mode`를 켜려면 **Postgres를 한 번 재시작**해야 하고, 몇 초 동안 API가 DB에 붙지 못합니다.
이미 인증된 NoaView PC는 로컬 머신 파일로 확인하므로 영향이 없습니다.

1. **RustFS 준비**
   - 버킷을 만듭니다. WAL-G는 버킷을 만들지 않습니다.
   - 백업 전용 사용자와 키를 만들고, 그 버킷 경로만 쓰게 합니다. 정책 예(버킷 `noaled-backup`, 경로 `keygen`):
     ```json
     {
       "Version": "2012-10-17",
       "Statement": [
         { "Effect": "Allow", "Action": ["s3:ListBucket"], "Resource": ["arn:aws:s3:::noaled-backup"],
           "Condition": { "StringLike": { "s3:prefix": ["keygen/*", "keygen"] } } },
         { "Effect": "Allow", "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
           "Resource": ["arn:aws:s3:::noaled-backup/keygen/*"] }
       ]
     }
     ```
   - `keygen-postgres`·`keygen-backup` 컨테이너에서 RustFS 주소(`BACKUP_S3_ENDPOINT`)에 닿는지 확인합니다. 두 서비스는 compose 기본 네트워크에만 있으므로, 외부 주소(https)를 쓰는 게 간단합니다.
2. **`.env.backup` 만들기**: `backup.env.example`을 복사해 채웁니다. 암호화 키는 `openssl rand -hex 32`로 만듭니다.
3. **암호화 키 따로 보관**: `BACKUP_ENCRYPTION_KEY`와 Keygen `.env`의 `ENCRYPTION_*`·`SECRET_KEY_BASE`를 비밀번호 관리자에 보관합니다. S3가 아닌 곳이어야 합니다. 이 값을 잃으면 백업이 있어도 복원할 수 없습니다.
4. **시크릿 등록**: Repository Secrets에 `DOT_ENV_BACKUP`(`base64 -w0 .env.backup`)을 추가합니다. 없으면 배포가 멈춥니다.
5. **만약을 위한 덤프** (서버에서):
   ```bash
   docker compose -f docker-compose.noaled.yaml exec -T keygen-postgres pg_dump -U <POSTGRES_USER> -Fc <POSTGRES_DB> > before-walg.dump
   ```
6. **배포**: **Deploy with Docker Compose** 워크플로를 실행합니다. 이미지를 받아 `keygen-postgres`를 다시 만들고 `keygen-backup`을 띄웁니다.
7. **확인**
   ```bash
   docker compose -f docker-compose.noaled.yaml logs keygen-backup      # "✅ … 백업 성공 (전체 …)"
   docker compose -f docker-compose.noaled.yaml exec keygen-backup walg backup-list
   docker compose -f docker-compose.noaled.yaml exec keygen-postgres \
     psql -U <POSTGRES_USER> -c "select archived_count, failed_count from pg_stat_archiver"
   ```
8. 1주일 안에 [복원 시험](#복원-시험-정기)을 한 번 합니다.

## 운영 명령

```bash
C="docker compose -f docker-compose.noaled.yaml"
$C exec keygen-backup keygen-backup full                       # 지금 전체 백업
$C exec keygen-backup keygen-backup delta                      # 지금 증분 백업
$C exec keygen-backup walg backup-list --detail                # 백업 목록
$C exec keygen-backup walg wal-verify integrity timeline       # WAL 연속성 검사
$C logs --tail=100 keygen-backup
```

## 복원 시험 (정기)

운영 DB는 건드리지 않습니다. 임시 볼륨에 복원해서 확인한 뒤 지웁니다.
복원한 DB는 `archive_mode=off`로 띄웁니다. 운영 저장소에 WAL을 올리지 않게 하기 위해서입니다.

```bash
C="docker compose -f docker-compose.noaled.yaml"
IMG=ghcr.io/noaled-lab/keygen-postgres:17.5-walg3.0.9
docker volume create keygen-restore-test
docker run --rm --env-file .env.backup -u postgres -v keygen-restore-test:/var/lib/postgresql/data $IMG keygen-restore
docker run -d --name keygen-restore-test --env-file .env.backup -e POSTGRES_PASSWORD=x \
  -v keygen-restore-test:/var/lib/postgresql/data $IMG postgres -c archive_mode=off
docker logs -f keygen-restore-test          # "database system is ready to accept connections"
docker exec -u postgres keygen-restore-test psql -U <POSTGRES_USER> -d <POSTGRES_DB> \
  -c "select count(*) from licenses" -c "select count(*) from machines"
docker rm -f keygen-restore-test && docker volume rm keygen-restore-test
```

숫자가 운영과 맞는지 봅니다. Keygen API까지 붙여서 서명이 되는지(머신 파일 발급) 확인하려면 [장애 복원](#장애-복원)처럼 compose로 띄웁니다.

## 장애 복원

1. API를 멈춥니다: `$C stop keygen-api keygen-worker keygen-backup`
2. Postgres를 멈추고, 기존 볼륨은 지우지 말고 이름을 바꿔 보관합니다. 또는 새 볼륨 이름을 compose에 지정합니다.
3. 빈 `postgres` 볼륨에 받습니다.
   ```bash
   $C run --rm --no-deps -u postgres --entrypoint keygen-restore keygen-postgres                       # 최신까지
   $C run --rm --no-deps -u postgres --entrypoint keygen-restore keygen-postgres LATEST "2026-11-26 14:30:00+09"   # 그 시각 직전까지
   ```
4. `$C up -d keygen-postgres` → 로그에서 복구가 끝났는지 확인 → `$C up -d`
5. 복원한 DB는 system_identifier가 같아서 백업이 그대로 이어집니다. 복원 직후 `keygen-backup full`을 한 번 합니다.

**시점 복원은 마지막으로 올라간 WAL까지만 됩니다.** 장애 직전 최대 5분(`archive_timeout`)은 잃을 수 있습니다.
복원에는 `.env.backup`(특히 `BACKUP_ENCRYPTION_KEY`)과 Keygen `.env`의 `ENCRYPTION_*`·`SECRET_KEY_BASE`가 필요합니다.

## 주의

- **한 경로(`BACKUP_S3_PATH`)에는 DB 하나만** 백업합니다. 다른 DB의 백업이 있는 경로면 `keygen-backup`이 실패하고 알립니다(system_identifier 비교). 같은 이름의 다른 WAL도 덮어쓰지 않습니다(`WALG_PREVENT_WAL_OVERWRITE`). 새로 만든 DB(`keygen:setup`)나 다른 서버는 새 경로를 씁니다.
- **WAL 업로드가 계속 실패하면 서버 디스크가 찹니다.** 매 백업마다 `pg_stat_archiver`를 확인해서, 실패 중이면 실패 알림을 보냅니다. 저장소 키가 만료되거나 주소가 바뀌면 바로 고칩니다.
- `BACKUP_ENCRYPTION_KEY`는 한 번 정하면 바꾸지 않습니다. 바꾸면 그 전 백업을 열 수 없습니다.
- 이미지 태그는 compose와 `build-postgres.yml`의 `TAG`에 고정되어 있습니다. Postgres나 WAL-G 버전을 바꿀 때는 둘을 함께 바꿉니다. Postgres **메이저** 버전 업그레이드(17 → 18)는 물리 백업으로 할 수 없습니다. 이때는 `pg_dump`로 옮기고, 새 경로로 백업을 새로 시작합니다.

## 트러블슈팅

| 증상 | 확인 |
| --- | --- |
| `walg: .env.backup에 값이 없습니다: …` | `.env.backup`의 필수 값(엔드포인트·버킷·키·암호화 키) |
| `failed to list s3 folder` / `AccessDenied` | 키·정책, `BACKUP_S3_ENDPOINT`에 컨테이너에서 닿는지, `BACKUP_S3_FORCE_PATH_STYLE=true` |
| `NoSuchBucket` | 버킷을 먼저 만들었는지 |
| `다른 DB의 백업이 있는 경로` | 위 [주의](#주의) 첫 항목. `BACKUP_S3_PATH`를 새 경로로 |
| `WAL 보관(archive_command)이 실패 중` | `$C logs keygen-postgres`에서 `archive command failed` 원인 |
| 알림이 안 옴 | `BACKUP_WEBHOOK_URL`, `BACKUP_WEBHOOK_FORMAT`. 알림 실패는 백업 결과에 영향 없이 로그에만 남음 |
