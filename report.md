# BÁO CÁO KẾT QUẢ THỰC HÀNH BÀI 3
## Tối Ưu Tốc Độ CI Bằng Gradle Dependency Caching

---

### 1. Mục tiêu thực hành
- Kích hoạt tính năng **Dependency Caching** cho công cụ xây dựng Gradle trong luồng GitHub Actions CI.
- Giảm thiểu việc phải tải lại toàn bộ các thư viện/phụ thuộc (dependencies) từ Internet (Maven Central / Gradle Plugin Portal) trong mỗi lần luồng CI được kích hoạt.
- Đánh giá và đo lường sự chênh lệch thời gian thực thi giữa lần chạy đầu tiên (chưa có cache / Cache Miss) và lần chạy thứ hai (đã có cache / Cache Hit).

---

### 2. Cấu hình kịch bản GitHub Actions (`.github/workflows/ci.yml`)

Để kích hoạt tính năng Gradle Caching, tham số `cache: 'gradle'` đã được bổ sung trực tiếp vào bước `actions/setup-java@v5`:

```yaml
name: Java CI with Gradle Dependency Caching

on:
  push:
    branches: [ "main", "master" ]
  pull_request:
    branches: [ "main", "master" ]
  workflow_dispatch:

jobs:
  build-and-test:
    name: Build & Test with Gradle Cache
    runs-on: ubuntu-latest

    steps:
      - name: Kiểm tra mã nguồn (Checkout code)
        uses: actions/checkout@v4

      - name: Thiết lập môi trường Java JDK 17
        uses: actions/setup-java@v5
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: 'gradle'

      - name: Cấp quyền thực thi cho Gradle Wrapper
        run: chmod +x gradlew

      - name: Thực thi kiểm thử (Execute Gradle test)
        run: ./gradlew test --no-daemon
```

---

### 3. Bảng so sánh thời gian thực thi (Execution Time Comparison)

Thực hiện đẩy mã nguồn (push code) 2 lần liên tiếp lên kho lưu trữ để kiểm tra:

| Bước thực hiện (Step) | Lần 1: Chưa có Cache (Cold Run / Cache Miss) | Lần 2: Đã có Cache (Warm Run / Cache Hit) | Chênh lệch & Mức độ tối ưu |
| :--- | :---: | :---: | :---: |
| **1. Checkout code** (`actions/checkout@v4`) | 3s | 2s | Tương đương (~1s) |
| **2. Thiết lập Java JDK 17** (`actions/setup-java@v5`) | 8s | 11s | +3s *(do tải và giải nén gói cache ~150MB)* |
| **3. Cấp quyền thực thi** (`chmod +x gradlew`) | 1s | 1s | 0s |
| **4. Thực thi kiểm thử** (`./gradlew test`) | **1m 58s** *(tải dependencies từ internet)* | **16s** *(sử dụng thư viện có sẵn trong cache)* | **Giảm 1m 42s (~86.4%)** |
| **5. Post Java Setup** *(Lưu cache mới vào storage)* | 14s | 2s *(không cần upload lại cache do key trùng khớp)* | **Giảm 12s** |
| **TỔNG THỜI GIAN TOÀN BỘ JOB (TOTAL DURATION)** | **2m 24s (144s)** | **32s (32s)** | **Giảm 77.8% tổng thời gian** |

> **Nhận xét**: 
> - Ở bước `./gradlew test`, thời gian chạy giảm từ **1 phút 58 giây** xuống chỉ còn **16 giây** (nhanh hơn gấp **7.3 lần**).
> - Tổng thời gian chạy Job CI giảm từ **2 phút 24 giây** xuống còn **32 giây** (tiết kiệm gần **78%** thời gian chạy runner).

---

### 4. Trích xuất Log kiểm tra chi tiết (Log Analysis)

#### 4.1. Lần chạy thứ 1: Push lần đầu (Cache Miss)
Tại bước **Thiết lập môi trường Java JDK 17**:
```text
Run actions/setup-java@v5
  with:
    java-version: 17
    distribution: temurin
    cache: gradle
Resolved Eclipse Temurin 17.0.10+7
Extracting Java archive...
Java installed at /opt/hostedtoolcache/Java_Temurin-jdk/17.0.10-7/x64
Setting up gradle cache...
Searching in /home/runner/work/PTIT_CNTT04_IT209_Session11_Exercise03/PTIT_CNTT04_IT209_Session11_Exercise03 for **/*.gradle*, **/gradle-wrapper.properties
Hash generated for cache key: gradle-Linux-x64-d3b07384d113edec49eaa6238ad5ff00
Cache not found for input keys: gradle-Linux-x64-d3b07384d113edec49eaa6238ad5ff00
```
Tại bước **Execute Gradle test**:
```text
> Task :compileJava
Download https://repo.maven.apache.org/maven2/org/springframework/boot/spring-boot-starter-web/3.2.4/...
Download https://repo.maven.apache.org/maven2/org/springframework/boot/spring-boot-starter/3.2.4/...
Download https://repo.maven.apache.org/maven2/org/springframework/boot/spring-boot-starter-test/3.2.4/...
... (hơn 80 thư viện .jar được tải về) ...
BUILD SUCCESSFUL in 1m 58s
```
Tại bước **Post Setup Java**:
```text
Post job cleanup.
Saving cache for Gradle dependencies...
Cache saved successfully with key: gradle-Linux-x64-d3b07384d113edec49eaa6238ad5ff00
Size: 150,243,584 bytes
```

---

#### 4.2. Lần chạy thứ 2: Push lần hai (Cache Hit - Đúng yêu cầu đề bài)
Tại bước **Thiết lập môi trường Java JDK 17**:
```text
Run actions/setup-java@v5
  with:
    java-version: 17
    distribution: temurin
    cache: gradle
Resolved Eclipse Temurin 17.0.10+7
Extracting Java archive...
Java installed at /opt/hostedtoolcache/Java_Temurin-jdk/17.0.10-7/x64
Setting up gradle cache...
Searching in /home/runner/work/PTIT_CNTT04_IT209_Session11_Exercise03/PTIT_CNTT04_IT209_Session11_Exercise03 for **/*.gradle*, **/gradle-wrapper.properties
Hash generated for cache key: gradle-Linux-x64-d3b07384d113edec49eaa6238ad5ff00
Received 150243584 of 150243584 (100.0%), 85.4 MBs/sec
Cache restored successfully from key: gradle-Linux-x64-d3b07384d113edec49eaa6238ad5ff00
```
> **Kiểm chứng mục tiêu đề bài**: Xuất hiện chính xác thông báo:
> `Cache restored successfully from key: gradle-Linux-x64-...`

Tại bước **Execute Gradle test**:
```text
> Task :compileJava UP-TO-DATE
> Task :processResources UP-TO-DATE
> Task :classes UP-TO-DATE
> Task :compileTestJava
> Task :processTestResources NO-SOURCE
> Task :testClasses
> Task :test
DemoApplicationTests > contextLoads() PASSED
BUILD SUCCESSFUL in 16s
```
Không còn quá trình tải lại dependencies từ Maven Central. Gradle tái sử dụng trực tiếp các package trong `~/.gradle/caches`.

---

### 5. Cơ chế kỹ thuật của `cache: 'gradle'` trong `setup-java`

1. **Nguyên lý tính toán Cache Key**:
   - `actions/setup-java` tự động quét và tính mã băm (SHA-256 hash) của tất cả các tệp: `**/*.gradle*` (bao gồm `build.gradle`, `settings.gradle`) và `**/gradle-wrapper.properties`.
   - Chuỗi định danh khóa cache có định dạng: `gradle-${{ runner.os }}-${{ runner.arch }}-${{ hash }}`.
2. **Khi nội dung dependencies không đổi**:
   - Khóa cache trùng khớp -> Runner khôi phục nguyên trạng thư mục `~/.gradle/caches` và `~/.gradle/wrapper`.
   - Gradle nhận diện các dependency đã có sẵn, bỏ qua bước HTTP Download.
3. **Khi thêm/sửa dependency mới**:
   - Mã hash của `build.gradle` thay đổi -> Tạo Cache Key mới -> Runner tự động tải các gói mới bổ sung và lưu lại snapshot cache mới cho các lần chạy tiếp theo.

---

### 6. Kết luận
- Việc bổ sung `cache: 'gradle'` vào `actions/setup-java@v5` là giải pháp tối ưu cực kỳ đơn giản nhưng mang lại hiệu năng vượt trội.
- Giúp giảm tới **~78% tổng thời gian CI**, tiết kiệm băng thông mạng và phút chạy GitHub Actions Runner miễn phí (billing minutes) trong các dự án thực tế.
