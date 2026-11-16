# SPDX-FileCopyrightText: Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0

# Share dependencies and source across the manager and REST builds.
FROM golang:1.27.4@sha256:5e8afd493a7124a8586e134d9b532731161d4b9becf9dee5a5956b107426f93d AS builder
ENV GOPROXY=https://proxy.golang.org|direct
ARG VERSION=dev
ARG TARGETOS
ARG TARGETARCH

WORKDIR /workspace/management-engine
# Copy the Go Modules manifests
COPY management-engine/go.mod management-engine/go.sum ./
COPY controller/api/go.mod controller/api/go.sum /workspace/controller/api/
# Cache dependencies separately so source edits do not download them again.
RUN go mod download

# Copy the Go source from the repository root (filtered by Dockerfile.dockerignore).
COPY management-engine/ ./
COPY controller/api/ /workspace/controller/api/

# Build for the requested platform; an unset TARGETARCH uses the builder architecture.
FROM builder AS manager-builder
RUN --mount=type=cache,target=/root/.cache/go-build CGO_ENABLED=0 GOOS=${TARGETOS:-linux} GOARCH=${TARGETARCH} go build -buildvcs=false -trimpath -ldflags "-X main.version=${VERSION} -s -w -buildid=" -o manager ./cmd

FROM builder AS tenant-api-builder
RUN --mount=type=cache,target=/root/.cache/go-build CGO_ENABLED=0 GOOS=${TARGETOS:-linux} GOARCH=${TARGETARCH} go build -buildvcs=false -trimpath -ldflags "-X main.version=${VERSION} -s -w -buildid=" -o tenant-api ./cmd/tenant-api

FROM builder AS operator-api-builder
RUN --mount=type=cache,target=/root/.cache/go-build CGO_ENABLED=0 GOOS=${TARGETOS:-linux} GOARCH=${TARGETARCH} go build -buildvcs=false -trimpath -ldflags "-X main.version=${VERSION} -s -w -buildid=" -o operator-api ./cmd/operator-api

# Each runtime image contains only its service binary.
# Refer to https://github.com/GoogleContainerTools/distroless for more details
FROM gcr.io/distroless/static:nonroot@sha256:aafe993665bf6200db38a02c11a9351e782a84b6a45e7f33b6695638c88e3931 AS runtime
WORKDIR /
USER 65532:65532

FROM runtime AS tenant-api
COPY --from=tenant-api-builder /workspace/management-engine/tenant-api .
ENTRYPOINT ["/tenant-api"]

FROM runtime AS operator-api
COPY --from=operator-api-builder /workspace/management-engine/operator-api .
ENTRYPOINT ["/operator-api"]

# Keep manager last so builds without --target preserve the existing manager image.
FROM runtime AS manager
COPY --from=manager-builder /workspace/management-engine/manager .
ENTRYPOINT ["/manager"]
