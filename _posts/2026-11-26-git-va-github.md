---
title: "Git và GitHub"
description: "Hướng dẫn nền tảng về Git và GitHub: 3 vùng làm việc, quy trình commit, làm việc với nhánh, đồng bộ remote, xử lý merge conflict và hoàn tác an toàn."
date: 2026-11-26 17:53:00 +0700
categories: [DevOps, Git & GitHub]
tags: [git, github, version-control, branching, devops]
pin: false
toc: true
---

## 1. Tổng quan

**Git** là một hệ thống quản lý phiên bản phân tán (*Distributed Version Control System — DVCS*) do Linus Torvalds phát triển vào năm 2005. Trong quá trình phát triển phần mềm, Git giúp lưu lại lịch sử các phiên bản mã nguồn theo thời gian để lập trình viên dễ dàng kiểm tra lại, khôi phục khi phát sinh lỗi và ghép code (*merge*) mượt mà khi nhiều người cùng làm việc trên một dự án.

**GitHub** là nền tảng lưu trữ mã nguồn trên đám mây (*Remote Repository*) hoạt động dựa trên nguyên lý của Git. Nhờ GitHub, cá nhân hoặc nhóm phát triển có thể sao lưu, đồng bộ và cộng tác trên cùng một dự án qua Internet. *(Lưu ý: GitHub được Microsoft mua lại vào năm 2018, còn bản thân Git là công cụ độc lập ra đời từ trước đó rất lâu.)*

---

## 2. Các thành phần chính

Để hiểu cách Git vận hành, cần nắm rõ 3 vùng làm việc trên máy cá nhân cùng với khái niệm nhánh và kho lưu trữ từ xa:

- **Working Directory (Thư mục làm việc)**: Là thư mục dự án thực tế trên máy tính nơi bạn trực tiếp tạo, sửa hoặc xóa các tệp mã nguồn. Mọi thay đổi ban đầu đều diễn ra tại đây nhưng chưa được Git ghi nhận vào lịch sử.
- **Staging Area (Vùng chuẩn bị / *Stage*)**: Đóng vai trò là bước đệm trung gian giữa thư mục làm việc và kho lưu trữ. Khi bạn chọn lọc những thay đổi cần lưu, chúng sẽ được đưa vào vùng này trước khi đóng gói thành một phiên bản chính thức.
- **Local Repository (Kho lưu trữ cục bộ)**: Nằm trong thư mục ẩn `.git` ngay trong dự án của bạn. Đây là nơi lưu giữ toàn bộ lịch sử các phiên bản (*commit*) đã được xác nhận trên máy, cho phép bạn tra cứu hoặc quay lại bất kỳ mốc thời gian nào.
- **Nhánh (*Branch*)**: Là một luồng phát triển độc lập tách ra từ mạch code chính (thường là `main` hoặc `master`). Nhánh giúp bạn thử nghiệm hoặc xây dựng tính năng mới mà không làm ảnh hưởng đến phần code đang chạy ổn định trên nhánh chính.
- **Remote Repository (Kho lưu trữ từ xa — GitHub)**: Là bản sao của kho lưu trữ được đặt trên máy chủ của GitHub. Các thành viên trong nhóm sẽ đẩy (*push*) code từ Local Repository lên đây và kéo (*pull*) thay đổi của người khác về máy mình.

---

## 3. Cách dùng

### Thiết lập ban đầu

Trước khi tạo phiên bản đầu tiên trên máy, bạn cần khai báo danh tính tác giả để Git gắn vào lịch sử mỗi lần lưu:

```bash
git config --global user.name "<Tên của bạn>"
git config --global user.email "<Email của bạn>"
```

Tiếp theo, tùy vào việc bạn bắt đầu dự án mới hay tham gia dự án đã có trên GitHub:

- **Khởi tạo dự án mới trên máy**: Chạy `git init` trong thư mục dự án để biến nó thành một Git repository (tạo thư mục `.git`). Nếu sau đó muốn đưa dự án lên một repository đã tạo sẵn trên GitHub, bạn cần liên kết hai kho bằng lệnh `git remote add origin <url>` và có thể kiểm tra lại đường dẫn liên kết bằng `git remote -v`.
- **Tải dự án có sẵn từ GitHub**: Chạy `git clone <url>` trong lần đầu tiên để tải toàn bộ mã nguồn và lịch sử repository về máy, lúc này kết nối `origin` đã được thiết lập sẵn.

Để tránh đưa nhầm các tệp không cần thiết hoặc nhạy cảm lên Git (như `node_modules/`, tệp build, `.env`, mật khẩu hay API key), hãy tạo một tệp tên `.gitignore` ở thư mục gốc của dự án và liệt kê tên các tệp hoặc thư mục đó vào bên trong.

### Quy trình lưu phiên bản hằng ngày

Khi làm việc trong Working Directory, bạn nên kiểm tra trạng thái hiện tại trước khi lưu để biết tệp nào đã sửa, tệp nào đang ở Staging Area và mình đang đứng ở nhánh nào:

```bash
git status
```

Sau khi code thêm hoặc sửa code và chắc chắn về các thay đổi, bạn thực hiện hai bước để lưu thành một phiên bản mới trong Local Repository:

1. **Đưa thay đổi vào Staging Area**: Chạy `git add .` để chọn toàn bộ các tệp đã thêm, sửa hoặc xóa vào vùng chuẩn bị.
2. **Đóng gói thành phiên bản chính thức**: Chạy `git commit -m "<mô tả thay đổi>"`. Luôn kèm cờ `-m` cùng mô tả ngắn gọn để ghi chú lý do thay đổi và tránh việc Git tự động mở trình soạn thảo dòng lệnh.

Khi cần xem lại danh sách lịch sử các phiên bản gọn nhẹ để lấy mã commit (*commit hash*), bạn dùng lệnh:

```bash
git log --oneline
```

Khi đã có mã commit từ `git log --oneline` và muốn quay lại phiên bản cũ, bạn chọn cách phù hợp với mục đích:

- **Chỉ xem thử hoặc chạy thử bản cũ**: Chạy `git switch --detach <mã-commit>` (hoặc `git checkout <mã-commit>`) để tạm thời đưa thư mục làm việc về đúng thời điểm đó. Khi xem xong, chạy `git switch main` (hoặc tên nhánh đang làm) để quay về hiện tại.
- **Hủy bỏ một commit bị lỗi nhưng vẫn giữ nguyên lịch sử chung**: Chạy `git revert <mã-commit>` để tạo một commit mới đảo ngược thay đổi của commit cũ (an toàn khi đã đẩy code lên GitHub — xem thêm ở mục *Hoàn tác*).
- **Đưa nhánh quay hẳn về một mốc cũ trên máy cá nhân**: Chạy `git reset --hard <mã-commit>` để đặt lại nhánh về đúng commit đó và xóa các thay đổi sau mốc này. *(Chỉ dùng khi chưa `git push` lên GitHub vì lệnh này viết lại lịch sử commit.)*

### Làm việc với nhánh (*Branch*)

Khi phát triển tính năng mới hoặc làm việc nhóm, bạn không nên viết code trực tiếp trên `main` mà hãy tách sang một nhánh riêng:

1. **Tạo và chuyển sang nhánh mới**: Chạy `git switch -c <tên-nhánh>` (hoặc `git checkout -b <tên-nhánh>`). Lưu ý rằng lệnh `git branch <tên-nhánh>` chỉ tạo nhánh chứ không tự chuyển sang nhánh đó, nên nếu dùng cách cũ bạn sẽ dễ commit nhầm lên nhánh hiện tại.
2. **Phát triển và lưu trên nhánh mới**: Viết code, chạy thử rồi thực hiện `git add .` và `git commit -m "..."` như bình thường.
3. **Quay về nhánh chính và ghép code**: Khi tính năng đã chạy ổn định, bạn bắt buộc phải quay về nhánh chính bằng `git switch main`, sau đó mới chạy `git merge <tên-nhánh>` để gộp toàn bộ thay đổi từ nhánh phụ vào `main`.

### Đồng bộ với GitHub (*Remote*)

Khi cộng tác cùng người khác, hãy cập nhật code mới nhất từ GitHub trước khi bắt đầu làm việc và đẩy phần việc của bạn lên khi hoàn tất:

- **Kiểm tra thay đổi mà chưa gộp ngay**: Dùng `git fetch` nếu bạn chỉ muốn tải thông tin các thay đổi mới từ GitHub về để xem trước mà chưa đụng vào code đang làm trên máy.
- **Cập nhật và gộp code mới về máy**: Dùng `git pull` trước khi bắt đầu code để vừa tải về (`fetch`) vừa gộp thẳng (`merge`) các thay đổi mới nhất từ GitHub vào nhánh hiện tại.
- **Đẩy code lên GitHub**: Sau khi hoàn tất các bước trên máy (`branch` → `add` → `commit` → `merge`), chạy `git push` để gửi các commit từ Local Repository lên GitHub.

### Xử lý xung đột và hoàn tác an toàn

**Giải quyết xung đột khi ghép code (*Merge Conflict*)**:
Khi hai người cùng sửa một đoạn code giống nhau rồi chạy `git merge` hoặc `git pull`, Git sẽ tạm dừng quá trình gộp và báo xung đột. Lúc này, bạn mở tệp bị báo lỗi, tìm các đoạn được đánh dấu bởi `<<<<<<<`, `=======`, `>>>>>>>`, chọn giữ lại phần code đúng, xóa toàn bộ các ký tự đánh dấu đó đi, rồi chạy `git add .` và `git commit` để hoàn tất quá trình ghép code.

**Hoàn tác thay đổi khi có nhầm lẫn**:
- **Hủy thay đổi chưa commit**: Nếu bạn sửa nhầm trong Working Directory và chưa lưu thành commit, chạy `git restore <tên-file>` để trả tệp về trạng thái ở lần commit gần nhất. Trường hợp đã lỡ chạy `git add`, hãy dùng `git restore --staged <tên-file>` để rút tệp ra khỏi Staging Area trước.
- **Hoàn tác một commit đã đẩy lên GitHub**: Khi một commit lỗi đã được `git push` lên kho chung, hãy dùng `git revert <mã-commit>` (lấy mã từ `git log --oneline`). Lệnh này tạo ra một commit mới đảo ngược tác dụng của commit cũ nên không làm thay đổi lịch sử chung của cả nhóm.

### Bảng tóm tắt nhanh các lệnh thường dùng

| Nhóm thao tác | Lệnh | Công dụng chính |
| :--- | :--- | :--- |
| **Cấu hình & Khởi tạo** | `git config --global ...` | Khai báo `user.name` và `user.email` trước lần commit đầu tiên |
| | `git init` / `git clone <url>` | Khởi tạo repo mới trên máy hoặc tải repo có sẵn từ GitHub |
| | `git remote add origin <url>` | Liên kết repo tạo bằng `git init` với repo trên GitHub (`git remote -v` để kiểm tra) |
| **Lưu phiên bản** | `git status` / `git log --oneline` | Xem trạng thái thay đổi hiện tại hoặc lịch sử commit rút gọn |
| | `git add .` | Đưa các thay đổi vào Staging Area |
| | `git commit -m "<mô tả>"` | Lưu thay đổi từ Staging Area thành một phiên bản mới |
| **Nhánh & Ghép code** | `git switch -c <tên-nhánh>` | Tạo nhánh mới và chuyển ngay sang nhánh đó |
| | `git switch main` | Quay về nhánh chính |
| | `git merge <tên-nhánh>` | Gộp code từ nhánh phụ vào nhánh hiện tại |
| **Đồng bộ Remote** | `git fetch` / `git pull` | Tải thay đổi để xem trước (`fetch`) hoặc tải và gộp thẳng vào nhánh (`pull`) |
| | `git push` | Đẩy các commit từ máy lên GitHub |
| **Hoàn tác** | `git restore <tên-file>` | Hủy thay đổi chưa commit (thêm `--staged` để bỏ khỏi Staging Area) |
| | `git revert <mã-commit>` | Tạo commit mới đảo ngược commit lỗi một cách an toàn cho repo chung |
