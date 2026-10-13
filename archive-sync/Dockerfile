# rclone + supercronic: a cron runner built for containers (logs to stdout,
# graceful SIGTERM, never starts a job while the previous run is still going).
ARG RCLONE_VERSION
FROM rclone/rclone:${RCLONE_VERSION}

# Bump the version and BOTH checksums together:
# https://github.com/aptible/supercronic/releases
ARG SUPERCRONIC_VERSION=v0.2.49
ARG SUPERCRONIC_SHA256_ARM64=02aa0cb229ba09050cba6638059dadb9eedc2276632ea43d6a57a2f8c1629dd5
ARG SUPERCRONIC_SHA256_AMD64=a53ae236602c7338aba3fbaff40bda6300eae3b9fedb8261eb06cfe3724430c1
ARG TARGETARCH

RUN set -eu; \
    case "${TARGETARCH}" in \
      arm64) sha256="${SUPERCRONIC_SHA256_ARM64}" ;; \
      amd64) sha256="${SUPERCRONIC_SHA256_AMD64}" ;; \
      *) echo "unsupported architecture: ${TARGETARCH}" >&2; exit 1 ;; \
    esac; \
    wget -q -O /usr/local/bin/supercronic \
      "https://github.com/aptible/supercronic/releases/download/${SUPERCRONIC_VERSION}/supercronic-linux-${TARGETARCH}"; \
    echo "${sha256}  /usr/local/bin/supercronic" | sha256sum -c -; \
    chmod +x /usr/local/bin/supercronic

ENTRYPOINT ["supercronic", "-passthrough-logs", "/etc/crontab"]
