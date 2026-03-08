# Local Dev Build for Selfhosting

Build and deploy the latest dev version as a Docker image on a NAS.

## Prerequisites

- Go, Node.js (>=20), Docker
- Frontend repo cloned alongside this repo: `../donetick-frontend`

## Build

```bash
# 1. Build frontend
cd ../donetick-frontend
npm install
npm run build-selfhosted

# 2. Copy frontend dist into backend
cd ../donetick
rm -rf frontend/dist
cp -r ../donetick-frontend/dist frontend/dist

# 3. Build Go binary (cross-compile for x86_64 Linux)
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build \
  -ldflags "-X donetick.com/core/config.Version=dev -X donetick.com/core/config.Commit=$(git rev-parse --short HEAD) -X donetick.com/core/config.BuildDate=$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  -o donetick .

# 4. Build Docker image
docker build -f Dockerfile.local -t donetick/donetick:dev .

# 5. Export image for upload
docker save donetick/donetick:dev -o donetick-dev.tar
```

The commit hash is visible via `GET /api/v1/resource` (`api_version`, `api_commit`).

## docker-compose.yaml for the NAS

```yaml
services:
  donetick:
    image: donetick/donetick:dev
    container_name: donetick
    restart: unless-stopped
    ports:
      - 2021:2021
    volumes:
      - ./data:/donetick-data
      - ./config:/config
    environment:
      - DT_ENV=selfhosted
      - DT_SQLITE_PATH=/donetick-data/donetick.db
      - TZ=Etc/UTC
      - PUID=1000
      - PGID=1000
    healthcheck:
      test: wget --no-verbose --tries=1 --spider http://localhost:2021/api/v1/health || exit 1
      start_period: 1m
      timeout: 5s
      interval: 1m
      retries: 3
```

## Cleanup

```bash
rm -f donetick donetick-dev.tar
git checkout -- frontend/dist
```
