# Keygen용 Postgres + WAL-G 백업 이미지.
# 공식 postgres 이미지에 공식 WAL-G 실행 파일과 예약 실행 도구(supercronic)만 더한다. 버전과 체크섬은 여기서 고정한다.
#   keygen-postgres: archive_command로 WAL을 S3(RustFS 등)에 올린다.
#   keygen-backup:   같은 이미지로 정해진 시각에 전체/증분 백업 · 보관 정리 · 검증 · 알림을 한다.
# 설정은 전부 .env.backup(BACKUP_*)으로 주입한다. 이 이미지에는 비밀값이 없다.
FROM postgres:17.5

ARG TARGETARCH
# postgres:17.5는 Debian bookworm(glibc 2.36)이라 Ubuntu 22.04용 WAL-G 실행 파일이 맞는다 (24.04용은 더 새 glibc를 요구한다).
ARG WALG_VERSION=v3.0.9
ARG WALG_SHA256_AMD64=d6795c663894d836ba20840bffe8970aed81bbe25d3151bd4951f6f9c362def9
ARG WALG_SHA256_ARM64=072eb4a4a119f817e1fa888c64a776b01e3453b2e0d967a67d149adb18fde652
ARG SUPERCRONIC_VERSION=v0.2.51
ARG SUPERCRONIC_SHA256_AMD64=cc232108374e8e2b537045f2d5eaa1229f619db314c3ba6a14d38b35dc0aa123
ARG SUPERCRONIC_SHA256_ARM64=4ae22576a78223d8ccdf5f3b3660a5cbb5288416bf0ca96afc4a36d195932b94

RUN set -eux; \
    apt-get update; \
    apt-get install -y --no-install-recommends ca-certificates curl jq; \
    rm -rf /var/lib/apt/lists/*; \
    case "${TARGETARCH:-amd64}" in \
      amd64) walg_arch=amd64;   walg_sha="$WALG_SHA256_AMD64"; cron_arch=amd64; cron_sha="$SUPERCRONIC_SHA256_AMD64" ;; \
      arm64) walg_arch=aarch64; walg_sha="$WALG_SHA256_ARM64"; cron_arch=arm64; cron_sha="$SUPERCRONIC_SHA256_ARM64" ;; \
      *) echo "지원하지 않는 아키텍처: $TARGETARCH" >&2; exit 1 ;; \
    esac; \
    curl -fsSL -o /usr/local/bin/wal-g "https://github.com/wal-g/wal-g/releases/download/${WALG_VERSION}/wal-g-pg-22.04-${walg_arch}"; \
    echo "${walg_sha}  /usr/local/bin/wal-g" | sha256sum -c -; \
    curl -fsSL -o /usr/local/bin/supercronic "https://github.com/aptible/supercronic/releases/download/${SUPERCRONIC_VERSION}/supercronic-linux-${cron_arch}"; \
    echo "${cron_sha}  /usr/local/bin/supercronic" | sha256sum -c -; \
    chmod 755 /usr/local/bin/wal-g /usr/local/bin/supercronic; \
    wal-g --version

COPY bin/ /usr/local/bin/
RUN chmod 755 /usr/local/bin/walg /usr/local/bin/keygen-backup /usr/local/bin/keygen-backup-schedule /usr/local/bin/keygen-restore
