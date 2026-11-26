---
title: "Tài liệu Docker cơ bản"
description: "Tổng hợp kiến thức nền tảng về Docker: so sánh Container với VM, các thành phần cốt lõi, Dockerfile tối ưu cache, Volume và Docker Compose."
date: 2026-11-26 17:52:00 +0700
categories: [DevOps, Docker & K8s]
tags: [docker, container, dockerfile, docker-compose, devops]
pin: false
toc: true
---

## 1. Tổng quan

Trước khi công nghệ container phổ biến, việc chuyển mã nguồn từ máy lập trình viên sang máy chủ (server) hoặc máy của thành viên khác thường đòi hỏi cài đặt thủ công hàng loạt thư viện, công cụ và cấu hình hệ thống. Chỉ một khác biệt nhỏ về phiên bản hay hệ điều hành cũng có thể dẫn đến lỗi *"trên máy tôi vẫn chạy bình thường"* (*it works on my machine*).

**Docker** (ra mắt năm 2013) giải quyết bài toán này bằng cách cung cấp một bộ công cụ hoàn chỉnh để đóng gói ứng dụng cùng toàn bộ thư viện và cấu hình phụ thuộc vào trong một đơn vị chuẩn hóa gọi là **Container**. Nhờ đó, ứng dụng có thể chạy nhất quán trên mọi môi trường, từ máy cá nhân (local) đến server production. Docker quản lý trọn vẹn vòng đời này: từ bước đóng gói mã nguồn thành **Image**, khởi tạo và vận hành **Container**, cho đến kết nối mạng (**Network**) và gắn lưu trữ dữ liệu (**Volume**).

### So sánh Docker Container và Máy ảo (VM)

Để hiểu lý do Docker gọn nhẹ hơn máy ảo truyền thống, hãy nhìn vào mô hình 3 tầng của một hệ thống: `Application` (Ứng dụng) → `OS Kernel` (Nhân hệ điều hành) → `Hardware` (Phần cứng).

| Tiêu chí | Máy ảo (Virtual Machine) | Docker Container |
| :--- | :--- | :--- |
| **Cơ chế cô lập** | Dùng **Hypervisor** để ảo hóa phần cứng | Cô lập ở tầng **Application** (tiến trình và userland) bằng `namespaces` và `cgroups` |
| **Hệ điều hành (Kernel)** | Mỗi VM mang theo một **OS Kernel riêng** | Dùng chung **OS Kernel** và phần cứng với máy host |
| **Tốc độ & Tài nguyên** | Nặng, khởi động chậm (tính bằng phút) | Rất nhẹ, khởi động nhanh (tính bằng giây) |
| **Khả năng tương thích OS** | Chạy được hệ điều hành khác kernel với máy host | Phụ thuộc vào kernel của host |

> **Lưu ý trên Windows và macOS:** Vì Linux container luôn cần Linux Kernel để vận hành, Docker Desktop trên Windows hoặc macOS sẽ chạy ngầm một máy ảo Linux nhẹ nhằm cung cấp kernel cho các container hoạt động.

---

## 2. Các thành phần chính

Hệ sinh thái Docker vận hành dựa trên sự phối hợp chặt chẽ giữa các thành phần cốt lõi sau:

- **Docker Image**: Là bản đóng gói tĩnh chứa mã nguồn ứng dụng cùng toàn bộ thư viện và môi trường phụ thuộc. Image ở trạng thái chỉ đọc (`read-only`) và bất biến (`immutable`), đảm bảo tính nhất quán khi di chuyển giữa các máy. Đây cũng là đối tượng được lưu trữ, chia sẻ và đẩy/kéo trên các **Container Registry** (như Docker Hub).
- **Docker Container**: Là thực thể đang chạy được tạo ra từ một Image. Mỗi container hoạt động như một tiến trình cô lập trong không gian riêng (`namespaces`, `cgroups`), có biến môi trường (`env`), cổng mạng (`port`), hệ thống tệp (filesystem) riêng và một lớp đọc-ghi tạm thời (`writable layer`) ở trên cùng. Mọi dữ liệu cần lưu trữ lâu dài (như cơ sở dữ liệu) cần được tách ra ngoài lớp tạm này thông qua **Volume**.
- **Dockerfile**: Là tệp văn bản chứa tập hợp các chỉ thị mô tả từng bước để tự động đóng gói ứng dụng thành một Docker Image.
- **Docker Compose**: Là công cụ cho phép khai báo và điều phối nhiều container chạy cùng lúc thông qua một tệp cấu hình YAML (`compose.yaml` hoặc `docker-compose.yaml`), thay thế cho việc phải gõ nhiều lệnh `docker run` dài dòng.
- **Volume & Bind Mount**: Là các cơ chế lưu trữ giúp dữ liệu tồn tại độc lập với vòng đời của container hoặc chia sẻ trực tiếp mã nguồn giữa máy host và container.
- **Docker Network**: Là mạng ảo nội bộ giúp các container giao tiếp an toàn với nhau (đặc biệt qua tên dịch vụ/container) và kết nối với bên ngoài.

---

## 3. Cách dùng

### Quy trình làm việc cốt lõi (Workflow)

Trong thực tế phát triển phần mềm, Docker thường đi theo chu trình 3 giai đoạn từ máy cá nhân đến server:

1. **Phát triển ở Local (Dev)**: Lập trình viên viết mã nguồn và kéo các Image phụ trợ cần thiết (như database, cache) về chạy thành container trên máy cá nhân, thường khởi chạy đồng bộ bằng `docker compose up`.
2. **Đóng gói và Chia sẻ (Build & Push)**: Viết `Dockerfile` để đóng gói ứng dụng thành Image bằng lệnh `docker build`. Sau khi đăng nhập Registry qua `docker login` và gắn tag đúng định dạng `<username>/<ten-image>:<tag>`, đẩy Image lên Docker Hub hoặc Private Registry bằng `docker push`.
3. **Triển khai (Deploy)**: Trên server đã cài đặt Docker Engine, kéo Image từ Registry về bằng `docker pull` và khởi chạy thành container phục vụ người dùng thông qua `docker run` hoặc `docker compose up -d`.

### Các lệnh CLI cơ bản

Khi thao tác hằng ngày với Image, Container và Network, bạn sẽ sử dụng nhóm lệnh `docker` theo các tình huống tương ứng dưới đây:

| Lệnh | Công dụng & Ngữ cảnh sử dụng |
| :--- | :--- |
| `docker pull <image>` | Tải một Image từ Registry (mặc định là Docker Hub) về máy local trước khi chạy hoặc kiểm tra. |
| `docker images` *(hoặc `docker image ls`)* | Liệt kê toàn bộ Image đang lưu trữ trên máy để kiểm tra phiên bản hoặc dung lượng. |
| `docker run -d <image>` | Tạo và khởi chạy một Container mới từ Image; thêm cờ `-d` (`--detach`) khi muốn container chạy ngầm. |
| `docker ps` | Xem danh sách các Container đang chạy; thêm cờ `-a` để hiển thị cả những container đã dừng. |
| `docker stop <id\|name>` / `docker start <id\|name>` | Dừng một Container đang chạy hoặc khởi động lại một Container đã dừng trước đó. |
| `docker logs <id\|name>` | Theo dõi nhật ký (log) đầu ra của Container, rất hữu ích khi kiểm tra hoạt động hoặc tìm nguyên nhân lỗi. |
| `docker exec -it <id\|name> bash` | Mở phiên shell tương tác bên trong Container đang chạy để kiểm tra file hoặc debug (với các image tối giản như `alpine` không có `bash`, hãy thay bằng `sh`). |
| `docker network create <ten-mang>` | Tạo một mạng ảo nội bộ (`user-defined bridge`) để các Container trong cùng mạng có thể kết nối và gọi nhau trực tiếp qua tên. |
| `docker push <user/image:tag>` | Đẩy Image đã đóng gói lên Docker Hub hoặc Private Registry để chia sẻ và triển khai. |
| `docker rm <id>` / `docker rmi <image>` | Xóa Container đã dừng hoặc xóa Image không còn sử dụng để giải phóng dung lượng. |
| `docker system prune -a` | Dọn dẹp nhanh toàn bộ container đã dừng, network thừa và các image không gắn với container nào. |

### Đóng gói ứng dụng với Dockerfile & Tối ưu Build Cache

Một `Dockerfile` cơ bản thường kết hợp các chỉ thị chính sau:

```dockerfile
# Chọn image gốc (nhiều official image như node, python mặc định dùng Debian; alpine là biến thể gọn nhẹ phổ biến)
FROM node:20-alpine

# Khai báo biến môi trường bên trong image và container
ENV NODE_ENV=production

# Sao chép file định nghĩa thư viện trước để tận dụng build cache
COPY package*.json ./

# Chạy lệnh trong lúc BUILD image để cài đặt thư viện/công cụ
RUN npm install

# Sao chép phần mã nguồn còn lại từ máy host vào image
COPY . .

# Chỉ định lệnh mặc định chạy khi KHỞI ĐỘNG container (không chạy lúc build)
CMD ["node", "index.js"]
```

Để quá trình build nhanh, gọn và an toàn, cần lưu ý hai kỹ thuật quan trọng:

- **Tối ưu Build Cache**: Đặt các bước ít thay đổi (như `COPY package*.json ./` và `RUN npm install`) lên trước `COPY . .`. Nhờ vậy, khi bạn chỉ sửa mã nguồn ứng dụng, Docker sẽ tái sử dụng lớp cache đã cài thư viện thay vì phải tải lại từ đầu.
- **Dùng tệp `.dockerignore`**: Liệt kê các thư mục nặng hoặc thông tin nhạy cảm (như `node_modules`, `.git`, `.env`) vào `.dockerignore` để lệnh `COPY . .` không chép nhầm rác hay bí mật vào trong Image.

Khi đã hoàn thiện `Dockerfile`, chạy lệnh sau để build Image (lưu ý bắt buộc phải có đường dẫn build context ở cuối lệnh, thường là dấu `.` đại diện cho thư mục hiện tại):

```bash
docker build -t my-app:1.0 .
```

### Lưu trữ dữ liệu: Named Volume và Bind Mount

Vì lớp ghi của container sẽ mất đi khi container bị xóa, Docker cung cấp hai cách gắn lưu trữ từ bên ngoài tùy theo mục đích:

- **Named Volume** (ví dụ: `-v db_data:/var/lib/postgresql/data`): Vùng lưu trữ do Docker trực tiếp quản lý, tách biệt hoàn toàn với cấu trúc thư mục host. Đây là lựa chọn chuẩn cho các dữ liệu cần độ bền vững cao như cơ sở dữ liệu ở môi trường thật.
- **Bind Mount** (ví dụ: `-v ./src:/app/src`): Ánh xạ trực tiếp một thư mục trên máy host vào bên trong container. Cách này đặc biệt phù hợp khi lập trình (dev) vì mọi thay đổi trên mã nguồn sẽ phản ánh ngay lập tức vào container (*hot-reload*).

### Quản lý đa container với Docker Compose

Thay vì chạy từng container riêng lẻ và gắn thủ công các tham số mạng, biến môi trường hay volume, bạn có thể khai báo toàn bộ hệ thống trong tệp `compose.yaml` (từ Compose v2+, thuộc tính `version:` đã bị loại bỏ):

```yaml
services:
  web:
    image: my-app:1.0
    ports:
      - "8080:80"
    environment:
      - NODE_ENV=production
  db:
    image: postgres:16
    volumes:
      - db_data:/var/lib/postgresql/data

volumes:
  db_data:
```

- Để khởi chạy toàn bộ dịch vụ ở chế độ chạy nền, dùng `docker compose up -d`. Nếu tệp cấu hình có tên khác mặc định, truyền thêm cờ `-f`:
  ```bash
  docker compose -f custom-compose.yaml up -d
  ```
- Khi muốn dừng và dọn dẹp toàn bộ container cùng network do Compose tạo ra, chạy `docker compose down` (dữ liệu trong Volume vẫn được giữ nguyên). Chỉ thêm cờ `-v` (`docker compose down -v`) khi bạn thực sự muốn xóa sạch cả dữ liệu trong Volume.

### Xử lý một số lỗi thường gặp

- **Xung đột cổng (`port is already allocated`)**: Khi chạy cờ `-p <host>:<container>`, nếu gặp thông báo này nghĩa là cổng ở vế trái (trên máy host) đang bị một ứng dụng khác chiếm dụng. Bạn chỉ cần đổi sang một cổng trống khác ở vế trái (ví dụ: `-p 8081:80`).
- **Container vừa khởi động đã tự tắt (`Exited`)**: Hãy chạy `docker logs <id|name>` để xem thông báo lỗi của ứng dụng, hoặc dùng `docker inspect <id|name>` để kiểm tra mã thoát (`ExitCode`) và cấu hình chi tiết của container.

---

## 4. Tài liệu tham khảo
- [Docker Tutorial for Beginners [FULL COURSE in 3 Hours]](https://www.youtube.com/watch?v=3c-iBn73dDE&t=136s) — Docker tutoriall full course của TechWorld with Nana
- [What is a container? (Docker Learn)](https://docs.docker.com/learn/content/what-is-a-container/) — Giải thích nền tảng về container, image và sự khác biệt với máy ảo.
- [Docker Engine Overview](https://docs.docker.com/engine/) — Tổng quan kiến trúc client-server của Docker Engine (`docker` CLI, REST API, `dockerd`).
