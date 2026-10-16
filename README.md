# DevOps Final – Đề 008: Quản lý thư viện

Bộ mã nguồn khởi đầu cho bài thi **DevOps Fundamentals – Cuối kỳ**.
Ứng dụng hiển thị **danh sách sách trong thư viện** lấy từ MySQL thông qua Spring Boot, trả về **dạng JSON** qua Nginx (reverse proxy).

## Kiến trúc

```
Trình duyệt ── http://<IP_máy_chủ>:<APP_PORT>
    ▼
[nginx]  nginx:1.27-alpine        GET /*  → app:8080
    ▼
[app]    Spring Boot 4 (JRE 21)   GET /api/books → mysql:3306
    ▼
[mysql]  mysql:8.0                volume mysql_data, KHÔNG public cổng mysql
```

## Cấu trúc thư mục

Đổi tên thư mục/repository thành `devops-final-de008-<họ-tên>` (ví dụ `devops-final-de008-nguyenvana`).
Dấu ★ là phần sinh viên phải làm.

```
devops-final-de008-nguyenvana/     ← Mã nguồn Spring Boot 4 (Java 21, Gradle) – CÓ SẴN
├── src/main/java/com/ptit/devops/library/   ← Controller, Entity, Repository
├── src/main/resources/application.properties
├── build.gradle                   ← Build ra file build/libs/app.jar
├── settings.gradle
├── Dockerfile                     ← ★ FILE KHUNG: tự hoàn chỉnh phần còn thiếu
├── .dockerignore                  ← CÓ SẴN
├── nginx/default.conf             ← ★ FILE KHUNG: tự đổi PORT (tránh trùng nhau)
├── database/init.sql              ← Bảng books + 8 bản ghi mẫu – CÓ SẴN
├── docker-compose.yml             ← ★ FILE KHUNG: mysql có sẵn, điền TODO 1–5
├── .github/workflows/ci-cd.yml    ← ★ FILE KHUNG: điền TODO 1–6
├── student.env                    ← ★ ĐIỀN THÔNG TIN SINH VIÊN
├── screenshots/                   ← ★ Ảnh minh chứng (Phần 5)
├── submission-content/            ← ★ Toàn bộ tệp minh chứng khi nộp bài
├── .gitignore                     ← CÓ SẴN
└── README.md
```

Phần **sinh viên phải làm** (xem đề thi):

| File | Việc cần làm |
|------|--------------|
| `student.env` | Điền thông tin sinh viên và thông số triển khai |
| `Dockerfile` | **Hoàn chỉnh**: multi-stage build, stage `build` (Gradle + JDK 21) và stage runtime (JRE 21) |
| `docker-compose.yml` | **Điền các TODO 1–5** (service `mysql` đã có sẵn) |
| `.github/workflows/ci-cd.yml` | **Điền các TODO 1–6** của pipeline build → push image → deploy |
| `screenshots/` | `01-actions.png`, `02-ghcr.png`, `03-compose-ps.png`, `04-website.png` |

## API của ứng dụng

| Method | Đường dẫn | Mô tả |
|--------|-----------|-------|
| GET | `/api/books` | Danh sách sách |

## Biến môi trường mà ứng dụng cần

| Biến | Ý nghĩa |
|------|---------|
| `DB_HOST`, `DB_PORT` | Địa chỉ MySQL (mặc định `mysql`, `3306`) |
| `DB_NAME`, `DB_USER`, `DB_PASSWORD` | Database và tài khoản kết nối |
| `APP_VERSION` | Phiên bản (truyền vào lúc build image, ví dụ commit SHA) |

## Chạy thử (khi đã hoàn thiện Dockerfile và docker-compose.yml)

```bash
docker compose --env-file student.env up -d --build
docker compose --env-file student.env ps
curl http://localhost:<APP_PORT>/api/books
docker compose --env-file student.env down
```

name: CI/CD
on:
push:
branches: [main]
workflow_dispatch:
env:
REGISTRY: ghcr.io

jobs:
build:
name: Build & Test
runs-on: ubuntu-latest
permissions:
contents: read
packages: write
outputs:
image: ${{ steps.meta.outputs.image }}
steps:
- uses: actions/checkout@v4
- name: Kiem tra student.env
run: |
set -a; source student.env; set +a
for v in STUDENT_NAME STUDENT_ID STUDENT_CLASS STUDENT_EMAIL GITHUB_USERNAME \
APP_PORT COMPOSE_PROJECT_NAME MYSQL_USER MYSQL_PASSWORD MYSQL_ROOT_PASSWORD; do
if [ -z "${!v}" ]; then exit 1; fi
done
if [ "$APP_PORT" -lt 8080 ]; then exit 1; fi
- name: Dat ten image
id: meta
run: echo "image=${REGISTRY}/${GITHUB_REPOSITORY,,}/app" >> "$GITHUB_OUTPUT"
# TODO 1: Khoi dong he thong bang docker compose up -d --build --wait --wait-timeout 300
- name: Build & chay thu
env:
APP_IMAGE: ${{ steps.meta.outputs.image }}
APP_TAG: ${{ github.sha }}
APP_VERSION: ${{ github.sha }}
run: docker compose --env-file student.env up -d --build --wait --wait-timeout 300
- name: Smoke test
run: |
set -a; source student.env; set +a
curl -fsS --retry 12 --retry-delay 5 --retry-all-errors "http://localhost:${APP_PORT}/api/labs" | grep -q "\"id\""
# TODO 2: Dang nhap GHCR bang docker/login-action@v3
- name: Dang nhap GHCR
uses: docker/login-action@v3
with:
registry: ${{ env.REGISTRY }}
username: ${{ github.actor }}
password: ${{ secrets.GITHUB_TOKEN }}
# TODO 3: Push image voi 2 tag: commit SHA va latest
- name: Push image len GHCR
env:
IMAGE: ${{ steps.meta.outputs.image }}
run: |
docker push "$IMAGE:${{ github.sha }}"
docker tag "$IMAGE:${{ github.sha }}" "$IMAGE:latest"
docker push "$IMAGE:latest"
- name: Don dep
if: always()
run: docker compose --env-file student.env down -v

deploy:
name: Deploy len server
# TODO 4: Chi chay khi job build thanh cong
needs: build
runs-on: ubuntu-latest
permissions:
contents: read
packages: read
steps:
- name: Deploy qua SSH
uses: appleboy/ssh-action@v1.2.0
env:
APP_IMAGE: ${{ needs.build.outputs.image }}
APP_TAG: ${{ github.sha }}
GHCR_USER: ${{ github.actor }}
GHCR_TOKEN: ${{ secrets.GITHUB_TOKEN }}
REPO_URL: ${{ github.server_url }}/${{ github.repository }}.git
DEPLOY_DIR: ${{ github.event.repository.name }}
with:
# TODO 5: Ket noi bang 3 secrets: SERVER_HOST, SERVER_USER, SERVER_SSH_KEY
host: ${{ secrets.SERVER_HOST }}
username: ${{ secrets.SERVER_USER }}
key: ${{ secrets.SERVER_SSH_KEY }}
envs: APP_IMAGE,APP_TAG,GHCR_USER,GHCR_TOKEN,REPO_URL,DEPLOY_DIR
script: |
set -e
if [ ! -d "$HOME/$DEPLOY_DIR/.git" ]; then git clone "$REPO_URL" "$HOME/$DEPLOY_DIR"; fi
cd "$HOME/$DEPLOY_DIR"
git fetch origin main && git reset --hard origin/main
echo "$GHCR_TOKEN" | docker login ghcr.io -u "$GHCR_USER" --password-stdin
export APP_IMAGE APP_TAG
# TODO 6: Keo image moi va khoi dong lai, khong build tren server
docker compose --env-file student.env pull app
docker compose --env-file student.env up -d --no-build --wait --wait-timeout 300
docker logout ghcr.io
docker compose --env-file student.env ps

