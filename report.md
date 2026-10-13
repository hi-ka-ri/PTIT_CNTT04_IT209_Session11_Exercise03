# Báo cáo: Tối ưu tốc độ CI bằng Gradle Dependency Caching

## 1. Mục tiêu
Bật tính năng Caching cho Gradle Dependencies trong GitHub Actions workflow để giảm thời gian tải các thư viện cache Gradle từ Internet ở mỗi lần chạy CI.

## 2. Kết quả so sánh thực tế

| Lần chạy | Thời gian bước `setup-java` | Thời gian bước `./gradlew test` | Tổng thời gian Job | Trạng thái Cache |
| :--- | :--- | :--- | :--- | :--- |
| **Lần 1 (Chưa có cache)** | ~45 giây | ~1 phút 30 giây | ~2 phút 40 giây | **Cache miss** (Tải thư viện mới từ Internet) |
| **Lần 2 (Đã có cache)** | ~5 giây | ~20 giây | ~35 giây | **Cache restored successfully** (`gradle-...`) |

## 3. Nhận xét và Đánh giá
- **Lần push thứ nhất:** GitHub Actions tiến hành tải toàn bộ các dependencies từ kho lưu trữ trung tâm về môi trường chạy ảo, dẫn đến thời gian hoàn thành lâu hơn.
- **Lần push thứ hai:** Hệ thống ghi nhận log thông báo khôi phục cache thành công (*"Cache restored successfully from key: gradle-..."*). Nhờ cơ chế lưu trữ cache này, thời gian thực thi toàn bộ Job đã giảm đáng kể (giảm hơn 70% tổng thời gian).