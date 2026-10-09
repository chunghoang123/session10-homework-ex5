# Session10 - EX5: Docker Compose Backend + PostgreSQL

## 1. Mục tiêu

Minh họa Infrastructure as Code (IaC) bằng Docker Compose v2:
định nghĩa toàn bộ stack (backend Node.js + database PostgreSQL,
network, volume, biến môi trường) trong file `docker-compose.yml`,
ai chạy `docker compose up` cũng ra môi trường giống hệt nhau.

App backend tái sử dụng ý tưởng ex2: đọc `DB_HOST` từ ENV,
`GET /` trả JSON, `GET /health` kiểm tra sức khỏe.
Không cần kết nối Postgres thật, chỉ đọc ENV để chứng minh
backend nhận diện được database qua network nội bộ.

## 2. Cấu trúc file

```text
ex5/
├── server.js          # app Node.js thuần (http), GET / và /health
├── package.json       # npm start -> node server.js
├── Dockerfile         # single-stage node:18-alpine
├── docker-compose.yml # Compose v2, không dùng `version:`
├── .env               # giá trị thật local (không commit)
├── .env.example       # mẫu, không chứa password thật
└── .gitignore         # node_modules/, .env
```

### server.js

- `DB_HOST = process.env.DB_HOST || 'not-set'`
- `GET /` → `{"message":"Hello from ex5 backend","db_host":"database"}`
- `GET /health` → `{"status":"ok"}`
- Lắng nghe port `3000` trong container
  (`process.env.PORT || 3000`; Compose không inject `PORT`
  vào container nên luôn là `3000`, host map qua `${PORT}:3000`).

### Dockerfile

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package.json ./
RUN npm install --omit=dev
COPY server.js ./
EXPOSE 3000
CMD ["node", "server.js"]
```

Single-stage, image nhẹ Alpine, `WORKDIR /app`,
`EXPOSE 3000` khớp `ports: "${PORT}:3000"`.

### docker-compose.yml (chuẩn v2, không có `version:`)

```yaml
services:
  database:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks:
      - backend-net

  backend:
    build: .
    ports:
      - "${PORT}:3000"
    environment:
      DB_HOST: database
    depends_on:
      - database
    networks:
      - backend-net

volumes:
  pgdata:

networks:
  backend-net:
```

### .env và .env.example

`.env` (giá trị thật local, đã cho vào `.gitignore`):

```ini
DB_NAME=appdb
DB_USER=appuser
DB_PASSWORD=apppass123
PORT=8081
```

`.env.example` (mẫu công khai, không password thật):

```ini
DB_NAME=appdb
DB_USER=appuser
DB_PASSWORD=dat-mat-khau-cua-ban-o-day
PORT=8080
```

Lưu ý: ban đầu `.env` để `PORT=8080` nhưng port này đã bị
container `app-single-stage` (bài khác) chiếm
(`0.0.0.0:8080->3000/tcp`), nên đổi sang `PORT=8081`.
Đây là cách xử lý port trùng tiêu chuẩn: không kill
container khác, đổi host port của mình.

## 3. Giải thích IaC: depends_on / networks / volumes

- **IaC (Infrastructure as Code):** toàn bộ hạ tầng nằm trong
  `docker-compose.yml` + `Dockerfile` + `.env`. Thay vì gõ lệnh
  `docker run` thủ công, chỉ cần `docker compose up -d --build`
  là dựng lại y hệt trên máy khác (miễn là có Docker Compose v2).
  File compose được version hóa bằng git, review được, tái lập được.

- **`depends_on: database`:** đảm bảo Compose khởi động
  `database` trước `backend`. Đây là thứ tự khởi động
  (service_started), không phải healthcheck — backend vẫn nên
  retry kết nối DB khi khởi động vì Postgres có thể chưa ready.

- **`networks: backend-net`:** tạo network bridge riêng `ex5_backend-net`.
  Hai service phân giải nhau bằng tên (`DB_HOST=database`
  chính là hostname của service `database` trong network này).
  Cô lập traffic khỏi các stack khác, không cần expose port 5432
  ra host.

- **`volumes: pgdata:/var/lib/postgresql/data`:** volume đặt tên
  `ex5_pgdata` lưu dữ liệu Postgres. Xóa container (`down`)
  không mất data; chỉ khi `down -v` mới xóa. Bài này demo
  `down` giữ volume nên data còn lại cho lần `up` sau.

## 4. Bảo mật .env

- `.env` chứa `DB_PASSWORD` thật → cho vào `.gitignore`,
  không bao giờ commit/push.
- Chỉ commit `.env.example` với password giả
  (`dat-mat-khau-cua-ban-o-day`) để người khác biết cần biến gì.
- Compose tự đọc `.env` để thay `${DB_NAME}`, `${DB_USER}`,
  `${DB_PASSWORD}`, `${PORT}` — không cần `export` thủ công.
- Không hardcode password trong `docker-compose.yml`,
  luôn đi qua biến môi trường.

## 5. Cách chạy (Windows PowerShell, không dùng sudo)

```powershell
cd C:\Users\chung\OneDrive\Desktop\devops\session10\ex5

# Kiểm tra file compose hợp lệ
docker compose config

# Dựng và chạy nền
docker compose up -d --build

# Xem trạng thái
docker compose ps

# Xem log (giới hạn 30 dòng, không prefix)
docker compose logs --no-log-prefix --tail 30

# Kiểm tra app
curl.exe -s http://localhost:8081/
curl.exe -s http://localhost:8081/health

# Dừng container, GIỮ volume/network data để demo
docker compose down

# (Không dùng ở đây) Xóa luôn volume:
# docker compose down -v
```

## 6. Kết quả kiểm chứng thực tế (09/10/2026)

Môi trường: Windows PowerShell, Docker 29.7.2,
Docker Compose v5.4.0.

Lần đầu `PORT=8080` thất bại vì trùng port:

```text
Error response from daemon: ports are not available:
exposing port TCP 0.0.0.0:8080 -> 127.0.0.1:0:
listen tcp 0.0.0.0:8080: bind: Only one usage of each
socket address (protocol/network address/port) is normally permitted.
```

Nguyên nhân: `docker ps` cho thấy
`app-single-stage  0.0.0.0:8080->3000/tcp` đang chiếm 8080.
Xử lý: đổi `.env` sang `PORT=8081`, chạy lại
`docker compose up -d --build` → thành công.

`docker compose ps`:

```text
NAME             IMAGE                SERVICE    STATUS        PORTS
ex5-backend-1    ex5-backend          backend    Up 4 seconds  0.0.0.0:8081->3000/tcp
ex5-database-1   postgres:15-alpine   database   Up 4 seconds  5432/tcp
```

`docker compose logs --no-log-prefix --tail 30`:

```text
PostgreSQL Database directory appears to contain a database; Skipping initialization
2026-10-09 03:51:30.760 UTC [1] LOG:  starting PostgreSQL 15.19 ...
2026-10-09 03:51:30.761 UTC [1] LOG:  listening on IPv4 address "0.0.0.0", port 5432
Backend listening on port 3000, DB_HOST=database
2026-10-09 03:51:30.812 UTC [1] LOG:  database system is ready to accept connections
```

`curl.exe -s http://localhost:8081/`:

```json
{"message":"Hello from ex5 backend","db_host":"database"}
```

`curl.exe -s http://localhost:8081/health`:

```json
{"status":"ok"}
```

Volume/network được tạo:

```text
NETWORK  ex5_backend-net  bridge
VOLUME   ex5_pgdata       local
```

Sau kiểm chứng chạy `docker compose down` (không `-v`)
để giữ volume `ex5_pgdata` cho lần demo sau.
