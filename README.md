# GoatSports Storage Service

Lưu trữ ảnh và tài liệu trên Cloudflare R2 (API tương thích S3) cho toàn hệ thống.

## Chức năng chính

- **Tải file lên** vào thư mục tạm, sau đó service nghiệp vụ yêu cầu chuyển sang thư mục chính thức khi dữ liệu đã lưu
  (luồng `file-move` qua Kafka, `FileMoveConsumer` xử lý rồi phát `file-move-completed`). Nhờ vậy file của thao tác
  bị hủy giữa chừng không trở thành file chính thức.
- **Phân quyền truy cập** (`FileAccessPolicy`): thư mục công khai (ảnh sân, avatar) và thư mục riêng tư
  (căn cước, giấy phép, ảnh chat). File riêng tư chỉ chủ sở hữu, admin, người có quyền duyệt hồ sơ, hoặc thành viên
  cuộc trò chuyện (hỏi social-service) mới đọc được, qua URL có chữ ký.
- **Giới hạn tải lên** theo thư mục (`UploadPolicy`: loại file, dung lượng).

## API chính

| Đường dẫn | Mô tả |
|---|---|
| `/api/v1/files` | Tải lên, lấy URL, xóa file |

## Cấu trúc

```
src/main/java/iuh/fit/storageservice/
├── application/     # service, port, DTO, chính sách truy cập
├── infrastructure/  # client S3 / R2, Kafka, Feign
└── presentation/    # REST controller
shared-kernel/       # @AuditLog, Feign interceptor dùng chung
```

## Tích hợp

- **Cloudflare R2** (`CLOUDFLARE_R2_*`). Ảnh công khai phục vụ qua CDN; thư mục riêng tư bị WAF chặn truy cập trực tiếp.
- **Kafka nhận**: `file-move`. **Kafka phát**: `file-move-completed`, `user-log`.
- **Feign**: social-service (kiểm tra quyền xem ảnh chat).

## Chạy và kiểm thử

```bash
mvn -f shared-kernel/pom.xml clean install -DskipTests
mvn spring-boot:run -Dspring-boot.run.profiles=dev     # cần application-dev.properties (bị .gitignore)
mvn test
```

## CI/CD

`.gitlab-ci.yml`: test → `jib:build` lên Docker Hub → gọi Jenkins job `GoatSports-CD`.
