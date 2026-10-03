# MinIO server and mc, built from source.
#
# Upstream's Dockerfile.release downloads signed binaries from dl.min.io,
# which now answers 410 Gone. This builds the same binaries from the release
# tags of our forks (acpvarchitects/minio, acpvarchitects/mc) and reproduces
# the upstream runtime layout: ubi-micro, minio + mc + static curl in
# /usr/bin, same entrypoint and environment. Stacks that use `mc ready local`
# or `curl -f .../minio/health/live` as healthcheck keep working.
#
#   docker build -f acpv/Dockerfile --target minio \
#     --build-arg RELEASE=RELEASE.2025-04-22T22-12-26Z .
#   docker build -f acpv/Dockerfile --target mc .
#
# The build context is not used: sources are cloned at the given tags.

ARG GO_IMAGE=golang:1.24-alpine
ARG UBI_IMAGE=registry.access.redhat.com/ubi9/ubi-micro:latest

FROM ${GO_IMAGE} AS build

ARG RELEASE=RELEASE.2025-04-22T22-12-26Z
ARG MC_RELEASE=RELEASE.2025-08-13T08-35-41Z
ARG TARGETARCH=amd64

ENV GOPATH=/go \
    CGO_ENABLED=0

RUN apk add --no-cache git bash curl ca-certificates

WORKDIR /src

RUN git clone --quiet --depth 1 --branch "${RELEASE}" https://github.com/acpvarchitects/minio.git minio && \
    git clone --quiet --depth 1 --branch "${MC_RELEASE}" https://github.com/acpvarchitects/mc.git mc

# Same ldflags as upstream `make hotfix`/release builds: gen-ldflags takes the
# release date as RFC3339 and the RELEASE prefix from the environment, so
# `minio --version` prints the upstream tag and the fork's commit id.
RUN to_rfc3339() { echo "$1" | sed 's#^RELEASE\.\([0-9-]*\)T\([0-9]*\)-\([0-9]*\)-\([0-9]*\)Z$#\1T\2:\3:\4Z#'; } && \
    cd /src/minio && \
    LDFLAGS="$(MINIO_RELEASE=RELEASE go run buildscripts/gen-ldflags.go "$(to_rfc3339 "${RELEASE}")")" && \
    go build -tags kqueue -trimpath -ldflags "${LDFLAGS}" -o /go/bin/minio && \
    cd /src/mc && \
    LDFLAGS="$(MC_RELEASE=RELEASE go run buildscripts/gen-ldflags.go "$(to_rfc3339 "${MC_RELEASE}")")" && \
    go build -tags kqueue -trimpath -ldflags "${LDFLAGS}" -o /go/bin/mc

RUN TARGETARCH="${TARGETARCH}" bash /src/minio/dockerscripts/download-static-curl.sh

# ---------------------------------------------------------------------------

FROM ${UBI_IMAGE} AS minio

ARG RELEASE

LABEL name="MinIO" \
      vendor="ACPV build of MinIO (AGPL-3.0)" \
      version="${RELEASE}" \
      release="${RELEASE}" \
      org.opencontainers.image.source="https://github.com/acpvarchitects/minio" \
      org.opencontainers.image.licenses="AGPL-3.0-only" \
      summary="MinIO object storage, built from source by ACPV after upstream stopped publishing images."

ENV MINIO_ACCESS_KEY_FILE=access_key \
    MINIO_SECRET_KEY_FILE=secret_key \
    MINIO_ROOT_USER_FILE=access_key \
    MINIO_ROOT_PASSWORD_FILE=secret_key \
    MINIO_KMS_SECRET_KEY_FILE=kms_master_key \
    MINIO_CONFIG_ENV_FILE=config.env \
    MC_CONFIG_DIR=/tmp/.mc

RUN chmod -R 777 /usr/bin

COPY --from=build /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=build /go/bin/minio /go/bin/mc /go/bin/curl /usr/bin/
COPY --from=build /src/minio/CREDITS /licenses/CREDITS
COPY --from=build /src/minio/LICENSE /licenses/LICENSE
COPY --from=build /src/minio/dockerscripts/docker-entrypoint.sh /usr/bin/docker-entrypoint.sh

EXPOSE 9000
VOLUME ["/data"]

ENTRYPOINT ["/usr/bin/docker-entrypoint.sh"]
CMD ["minio"]

# ---------------------------------------------------------------------------

FROM ${UBI_IMAGE} AS mc

ARG MC_RELEASE

LABEL name="MinIO Client" \
      vendor="ACPV build of mc (AGPL-3.0)" \
      version="${MC_RELEASE}" \
      release="${MC_RELEASE}" \
      org.opencontainers.image.source="https://github.com/acpvarchitects/mc" \
      org.opencontainers.image.licenses="AGPL-3.0-only"

ENV MC_CONFIG_DIR=/tmp/.mc

COPY --from=build /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=build /go/bin/mc /usr/bin/mc
COPY --from=build /src/mc/CREDITS /licenses/CREDITS
COPY --from=build /src/mc/LICENSE /licenses/LICENSE

ENTRYPOINT ["mc"]
