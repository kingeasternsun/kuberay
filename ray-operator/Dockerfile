# Volcano-style: module download on BUILDPLATFORM (fast) + GOPROXY; compile on the target platform
# so CGO-enabled manager builds against the correct libc/toolchain. Runtime stages use TARGETPLATFORM.

ARG GOLANG_IMAGE=golang:1.25-bookworm

FROM --platform=$BUILDPLATFORM ${GOLANG_IMAGE} AS deps
ARG GOPROXY=https://goproxy.cn,direct
ENV GO111MODULE=on
ENV GOPROXY=${GOPROXY}
WORKDIR /workspace
COPY go.mod go.sum ./
RUN go mod download

ARG GOLANG_IMAGE=golang:1.25-bookworm
FROM ${GOLANG_IMAGE} AS builder
ARG TARGETOS=linux
ARG TARGETARCH
ARG GOPROXY=https://goproxy.cn,direct
ENV GO111MODULE=on
ENV GOPROXY=${GOPROXY}
WORKDIR /workspace
COPY --from=deps /go/pkg/mod /go/pkg/mod
COPY go.mod go.sum ./
COPY main.go main.go
COPY apis/ apis/
COPY controllers/ controllers/
COPY pkg/features pkg/features
COPY pkg/utils pkg/utils
COPY pkg/webhooks pkg/webhooks
COPY rayjob-submitter/ rayjob-submitter/

USER root
RUN ARCH="${TARGETARCH:-$(go env GOARCH)}" \
    && CGO_ENABLED=1 GOOS=${TARGETOS} GOARCH=${ARCH} go build -tags strictfipsruntime -a -o manager main.go \
    && go version -m manager | awk -v arch="${ARCH}" '$1=="build" && $2=="GOARCH=" arch {found=1} END {if (!found) exit 1}'
RUN ARCH="${TARGETARCH:-$(go env GOARCH)}" \
    && CGO_ENABLED=0 GOOS=${TARGETOS} GOARCH=${ARCH} go build -ldflags="-w -s" -a -o submitter ./rayjob-submitter/cmd/main.go \
    && go version -m submitter | awk -v arch="${ARCH}" '$1=="build" && $2=="GOARCH=" arch {found=1} END {if (!found) exit 1}'

FROM --platform=$TARGETPLATFORM scratch AS submitter
WORKDIR /
COPY --from=builder /workspace/submitter .
USER 65532:65532

ENTRYPOINT ["/submitter"]

FROM --platform=$TARGETPLATFORM swr.cn-south-1.myhuaweicloud.com/ascendhub/ubuntu:24.04-base
WORKDIR /
COPY --from=builder /workspace/manager .
USER 65532:65532

ENTRYPOINT ["/manager"]
