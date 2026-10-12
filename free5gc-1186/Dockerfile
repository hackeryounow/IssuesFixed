# syntax=docker/dockerfile:1
#
# Packages the Session Management Function (SMF) from this bundle's source
# snapshot in pkgs/smf (upstream github.com/free5gc/smf @
# 1c9d7662fb51de39df0591bad9c609ac85fec64d, the exact revision the stock
# free5gc/smf:v4.3.0 image is built from - `go version -m` on its binary
# reports COMMIT_HASH=1c9d7662) with the fix for free5gc issue #1186 applied
# (pkgs/fix.patch, commit 669d389a in pkgs/smf).
#
# Structure follows /root/5gc/free5gc-compose:
#   base compile stage    -> Go toolchain building the module
#   nf_smf/Dockerfile     -> runtime stage (alpine:3.15, /free5gc layout, EXPOSE 8000)
#
# Deviations from the compose build, all forced by building the standalone NF
# repository instead of the free5gc monorepo:
#   1. the toolchain is golang:1.26.2, because pkgs/smf/go.mod declares `go 1.26.2`;
#   2. the compile command is `go build ./cmd`, because the standalone repository
#      carries no Makefile (the monorepo runs `make smf` for the same main package);
#   3. CGO is disabled: the SMF is pure Go (gin, mongo driver, pfcp/nas codecs)
#      and needs none of the libmnl/libyaml build deps of the UPF.
#
# pkgs/fix.patch is authored against 1c9d7662. Re-porting it to a newer release
# means re-checking the switch in HandlePDUSessionSMContextLocalRelease
# (pdu_session.go) and the guard at the top of RemoveSMContextFromAllNF
# (sm_common.go).
#
# Default certificates are NOT baked in: the deployment bind-mounts ./cert over
# /free5gc/cert, exactly as the stock deployment does.
#
# Build (context = this directory):
#   docker build -f Dockerfile -t free5gc-smf:1c9d766-1186 .
# Publish:
#   docker tag free5gc-smf:1c9d766-1186 swr.cn-north-4.myhuaweicloud.com/cn_5gc/free5gc-smf:1c9d766-1186
#   docker push swr.cn-north-4.myhuaweicloud.com/cn_5gc/free5gc-smf:1c9d766-1186

FROM golang:1.26.2-bookworm AS builder

ENV DEBIAN_FRONTEND=noninteractive

WORKDIR /go/src/free5gc/NFs/smf

# Resolve dependencies first so the compile layer is re-run on source-only edits.
COPY pkgs/smf/go.mod pkgs/smf/go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download

COPY pkgs/smf/ ./

RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 go build -trimpath -ldflags="-w -s" -o /go/bin/smf ./cmd \
    && go version -m /go/bin/smf | head -3

FROM alpine:3.15

# Revision baked into the provenance label; override with
# --build-arg REVISION=$(git -C pkgs/smf rev-parse HEAD) when building from a
# different commit of pkgs/smf.
ARG REVISION=1c9d7662fb51de39df0591bad9c609ac85fec64d+fix

LABEL description="Free5GC SMF built from this bundle's source snapshot (free5gc/smf @ 1c9d7662) with the fix for issue #1186" \
    version="v4.3.0-1186fix" \
    org.opencontainers.image.source="https://github.com/free5gc/smf" \
    org.opencontainers.image.revision="${REVISION}"

ARG DEBUG_TOOLS

ENV F5GC_MODULE=smf

# Install debug tools ~ 100MB (if DEBUG_TOOLS is set to true)
RUN if [ "$DEBUG_TOOLS" = "true" ] ; then apk add -U vim strace net-tools curl netcat-openbsd ; fi

# Set working dir
WORKDIR /free5gc
RUN mkdir -p config/ log/ cert/

# Copy executable (certs and configs come from the deployment's bind mounts)
COPY --from=builder /go/bin/smf ./

# Config files volume
VOLUME [ "/free5gc/config" ]

# Certificates (if not using default) volume
VOLUME [ "/free5gc/cert" ]

# Exposed ports
EXPOSE 8000

# No ENTRYPOINT, matching the stock free5gc/smf image: the deployment supplies
# the whole command line, e.g. `./smf -c ./config/smfcfg.yaml -u ./config/uerouting.yaml`.
