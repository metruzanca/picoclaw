# Building PicoClaw for ARMv6 (Raspberry Pi Zero W)

These steps cross-compile the Go binary on an x86\_64 host and build a
Docker image for `linux/arm/v6`.  The final image is then piped directly
to the Pi over SSH.

## Prerequisites

- Host with Go 1.25+, Docker, and Docker buildx
- Pi running Docker, reachable via SSH key (no password)

## Commands

### 1. Cross-compile the Go binary

```bash
VERSION=$(git describe --tags --always --dirty 2>/dev/null || echo dev)
GIT_COMMIT=$(git rev-parse --short=8 HEAD 2>/dev/null || echo dev)
BUILD_TIME=$(date +%FT%T%z)
GO_VERSION=$(go env GOVERSION 2>/dev/null || echo unknown)
CONFIG_PKG=github.com/sipeed/picoclaw/pkg/config
LDFLAGS="-X ${CONFIG_PKG}.Version=${VERSION} \
  -X ${CONFIG_PKG}.GitCommit=${GIT_COMMIT} \
  -X ${CONFIG_PKG}.BuildTime=${BUILD_TIME} \
  -X ${CONFIG_PKG}.GoVersion=${GO_VERSION} \
  -s -w"

mkdir -p build-armv6
GOOS=linux GOARCH=arm GOARM=6 CGO_ENABLED=0 go build \
  -v -tags goolm,stdjson \
  -ldflags "$LDFLAGS" \
  -o build-armv6/picoclaw ./cmd/picoclaw
```

### 2. Build the Docker image (ARMv6)

```bash
# The Dockerfile copies the binary from the build directory.
# Must be invoked from the project root so COPY paths resolve.
cp build-armv6/picoclaw docker/picoclaw
docker buildx build --platform linux/arm/v6 \
  -t docker.io/sipeed/picoclaw:latest \
  -f docker/Dockerfile.armv6 \
  . --load
rm docker/picoclaw
```

### 3. Transfer to the Raspberry Pi

```bash
docker save docker.io/sipeed/picoclaw:latest \
  | gzip \
  | ssh metru@192.168.1.160 "gunzip | docker load"
```

### 4. Run on the Pi

```bash
ssh metru@192.168.1.160
cd /path/to/picoclaw
docker compose -f docker/docker-compose.yml --profile gateway up
```

## Cleanup

```bash
rm -rf build-armv6
```
