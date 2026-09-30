# Bài 5: Phân tích và khắc phục `UnsupportedClassVersionError`

## 1. Phân tích nguyên nhân

`UnsupportedClassVersionError` xuất hiện khi một Java Virtual Machine (JVM) cố
nạp một file `.class` có định dạng mới hơn định dạng mà JVM đó hỗ trợ. Lỗi có
thể xảy ra trong lúc chạy ứng dụng, chạy test, hoặc ngay trong bước build nếu
Gradle, plugin hay dependency được nạp bằng một JVM quá cũ.

Hai số trong thông báo lỗi là phiên bản định dạng class file:

| Class file version | Phiên bản Java tối thiểu có thể chạy class đó |
| --- | --- |
| `61.0` | Java 17 |
| `65.0` | Java 21 |

Vì vậy, thông báo “compiled by a more recent version ... `65.0`, ... recognizes
versions up to `61.0`” cho biết JVM đang chạy chỉ hiểu class file đến Java 17,
nhưng class cần nạp đã được biên dịch cho Java 21. Java 17 không thể chạy class
file Java 21; cần nâng JVM lên Java 21 hoặc biên dịch class xuống một mức tương
thích nếu mã nguồn và các dependency cho phép.

Bảng đầy đủ có thể tra trong **Java Virtual Machine Specification**, mục
[The `ClassFile` Structure](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-4.html#jvms-4.1).
Mục này định nghĩa `major_version` và các phiên bản class file mà JVM chấp nhận.
Để đối chiếu release Java với class file, có thể kiểm tra các phiên bản của
JVM Specification tương ứng; ví dụ đặc tả Java 17 ghi phiên bản class tối đa
là `61`, còn đặc tả Java 21 ghi phiên bản tối đa là `65`.

Trong tình huống này, developer build bằng Java 21 nên class đầu ra có thể dùng
định dạng Java 21 (`65.0`), trong khi pipeline lại chạy build/test hoặc ứng
dụng bằng Java 17 (`61.0`). Lỗi không nhất thiết chỉ xảy ra khi biên dịch mã
nguồn chính: một plugin Gradle, dependency, test, hoặc class đã build sẵn cũng
có thể là class đầu tiên mà JVM cũ cố nạp.

Máy cá nhân và GitHub Actions là hai môi trường độc lập. Máy developer có thể
có JDK 21 được chọn sẵn trong IDE, terminal hoặc cấu hình Gradle. Runner CI có
thể dùng JDK khác; phiên bản mặc định của runner không được đảm bảo trùng với
máy cá nhân. Bước `actions/setup-java` trong workflow thiết lập JDK được yêu
cầu cho môi trường job thông qua `JAVA_HOME` và `PATH`. Ngoài ra, Gradle
toolchain và cấu hình target bytecode của dự án cũng cần được kiểm tra: chúng
quyết định compiler/định dạng class đầu ra và không nên mâu thuẫn với phiên
bản Java mà pipeline dự kiến hỗ trợ.

## 2. Vị trí cần kiểm tra và cách khắc phục

Rà soát bước `actions/setup-java` trong `.github/workflows/ci.yml`, đặc biệt
là `java-version`. Nếu pipeline phải build và chạy mã Java 21, chọn JDK 21
trên CI:

```yaml
steps:
  - name: Checkout code
    uses: actions/checkout@v7

  - name: Set up JDK 21
    uses: actions/setup-java@v4
    with:
      distribution: temurin
      java-version: '21'

  - name: Build with Gradle
    run: ./gradlew clean build
```

Theo tài liệu chính thức của
[`actions/setup-java`](https://github.com/actions/setup-java), action này cài
phiên bản Java được yêu cầu và cấu hình `JAVA_HOME` cùng `PATH` cho các bước
tiếp theo. Sau khi thay đổi, nên kiểm tra phiên bản thực tế mà job và Gradle
đang dùng, ví dụ chạy `java -version` và `./gradlew --version` trong CI.

Giữ cấu hình Gradle đồng nhất với quyết định về phiên bản:

```groovy
java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}
```

Nếu yêu cầu triển khai là phải chạy trên Java 17 thì không nên chỉ nâng runtime
CI lên 21 một cách máy móc. Thay vào đó, đặt compiler/toolchain và target
bytecode về Java 17 (ví dụ dùng `options.release = 17`), đồng thời xác nhận
mã nguồn và toàn bộ dependency tương thích Java 17. Mục
[Java toolchains và `--release`](https://docs.gradle.org/current/userguide/toolchains.html)
của Gradle giải thích cách cấu hình compiler và mức tương thích của bytecode.

## 3. Nguyên tắc phòng tránh tái diễn

1. Chốt phiên bản Java mà dự án hỗ trợ và ghi rõ trong tài liệu/configuration.
2. Cấu hình cùng phiên bản cho local, CI, Gradle toolchain và môi trường
   runtime; nếu cần chạy trên Java cũ hơn, đặt rõ target bytecode tương thích.
3. Đặt phiên bản Java cụ thể trong workflow, không dựa vào JDK mặc định có
   sẵn trên runner.
4. Trong CI, ghi nhận `java -version` và `./gradlew --version` để dễ phát hiện
   khác biệt môi trường; chạy build/test sạch để kiểm chứng thay đổi.
5. Khi nâng JDK, kiểm tra cả plugin và dependency vì chúng cũng chứa class file
   cần được JVM nạp.

### Tài liệu tham khảo

- [Oracle: Java Virtual Machine Specification, Java SE 21, §4.1](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-4.html#jvms-4.1)
- [Oracle: Java Virtual Machine Specification, Java SE 17](https://docs.oracle.com/javase/specs/jvms/se17/html/index.html)
- [GitHub: `actions/setup-java`](https://github.com/actions/setup-java)
- [Gradle: Toolchains for JVM projects](https://docs.gradle.org/current/userguide/toolchains.html)
