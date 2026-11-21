# GoogleUnlimited

> Tiện ích đồng bộ ảnh và video đa nền tảng tự động dành cho Google
> Photos. Chạy như một tiến trình nền gọn nhẹ trên Windows, macOS và
> Linux để tự động đồng bộ ảnh và video bằng cách giả lập thiết bị
> Google Pixel, sử dụng
> [`xob0t/gotohp`](https://github.com/xob0t/gotohp) --- không cần máy
> ảo, trình giả lập Android, Windows Subsystem for Android (WSA) hoặc
> WSL.
<p align="center">
  <img src="reddit_showcase.jpg" alt="GoogleUnlimited Showcase" width="100%" />
</p>
------------------------------------------------------------------------

## Mục lục

-   [Tổng quan](#tổng-quan)
-   [Tính năng](#tính-năng)
-   [Yêu cầu](#yêu-cầu)
-   [Bắt đầu nhanh](#bắt-đầu-nhanh)
    -   [Windows](#bắt-đầu-nhanh-trên-windows)
    -   [macOS](#bắt-đầu-nhanh-trên-macos)
    -   [Linux](#bắt-đầu-nhanh-trên-linux)
-   [Xác thực và thiết lập tài khoản](#xác-thực-và-thiết-lập-tài-khoản)
-   [Tham khảo lệnh](#tham-khảo-lệnh)
-   [Giám sát thư mục và định dạng phương tiện được hỗ
    trợ](#giám-sát-thư-mục-và-định-dạng-phương-tiện-được-hỗ-trợ)
-   [Chạy nền](#chạy-nền)
-   [Lưu trữ dữ liệu, quyền riêng tư và bảo
    mật](#lưu-trữ-dữ-liệu-quyền-riêng-tư-và-bảo-mật)
-   [Khắc phục sự cố](#khắc-phục-sự-cố)
-   [Giới hạn và tuyên bố miễn trừ trách
    nhiệm](#giới-hạn-và-tuyên-bố-miễn-trừ-trách-nhiệm)
-   [Ghi công và giấy phép](#ghi-công-và-giấy-phép)

------------------------------------------------------------------------

## Tổng quan

**GoogleUnlimited** giám sát các thư mục trên máy tính và tự động đồng
bộ những ảnh, video đủ điều kiện lên thư viện Google Photos. Công cụ kết
hợp bộ theo dõi sự kiện hệ thống tập tin gốc của từng hệ điều hành
(PowerShell trên Windows, `fswatch` trên macOS và `inotify-tools` trên
Linux), cơ sở dữ liệu SQLite nhúng (`gphotos.db`) và chương trình dòng
lệnh độc lập [`gotohp`](https://github.com/xob0t/gotohp).

Bộ máy tải lên giao tiếp trực tiếp với các điểm cuối tải lên dành cho
thiết bị di động của Google bằng cách giả lập thiết bị Google Pixel. Vì
các yêu cầu được gửi đi dưới danh nghĩa thiết bị Pixel, phương tiện được
sao lưu theo cơ chế áp dụng cho Pixel trong Google Photos.

------------------------------------------------------------------------

## Tính năng

-   **Giám sát hệ thống tập tin theo thời gian thực:**
    -   **Windows:** Sử dụng .NET `FileSystemWatcher`, bộ đệm sự kiện mở
        rộng 64 KB và khoảng chờ chống kích hoạt lặp 1,5 giây để tránh
        tải lên trùng khi tập tin được ghi liên tục.
    -   **macOS:** Sử dụng `fswatch` để liên tục nhận sự kiện hệ thống
        tập tin với độ trễ thấp.
    -   **Linux:** Sử dụng `inotifywait` (`inotify-tools`) để lắng nghe
        các sự kiện nhân hệ điều hành `close_write`, `moved_to` và
        `create`.
-   **Đồng bộ ban đầu và quét kiểm tra định kỳ:**
    -   Khi khởi động, công cụ quét toàn bộ thư mục được giám sát (Giai
        đoạn 1) để tải lên các tệp phương tiện hiện có nhưng chưa được
        ghi nhận.
    -   Chạy các đợt quét kiểm tra nền định kỳ: mỗi 30 giây trên Windows
        và mỗi 15 phút trên macOS/Linux. Việc này giúp phát hiện ảnh
        nhập từ máy ảnh khi công cụ ngoại tuyến hoặc tệp được thêm vào
        lúc bộ theo dõi tạm dừng.
-   **Chống tải lên trùng bằng mã băm nội dung đa nền tảng:**
    -   Theo dõi tất cả tệp đã tải lên trong cơ sở dữ liệu SQLite cục bộ
        `gphotos.db`, sử dụng chế độ WAL.
    -   Tra cứu dựa trên cả đường dẫn tệp và khóa kết hợp
        `filename|filesize`, đi kèm `content_hash` SHA-256. Nhờ vậy,
        công cụ tránh tải lên trùng trên máy tính khởi động kép hoặc ổ
        đĩa ngoài dùng chung, nơi đường dẫn có thể khác nhau giữa các hệ
        điều hành.
-   **Phát hiện tệp đang tải xuống hoặc bị khóa:**
    -   Kiểm tra tính ổn định của dung lượng tệp qua nhiều lần lấy mẫu
        và thử quyền ghi độc quyền trước khi tải lên. Điều này ngăn việc
        tải lên các tệp chưa hoàn chỉnh trong lúc trình duyệt tải xuống,
        OBS/Xbox Game Bar ghi màn hình hoặc phần mềm xuất video.
-   **Hàng đợi tự động thử lại:**
    -   Các lần tải lên thất bại được lưu trong bảng `retry_queue` lâu
        dài và tự động thử lại mỗi 30 giây, tối đa 3 lần trước khi ghi
        nhận lỗi vĩnh viễn.
-   **Tự động khởi chạy cùng hệ thống:**
    -   **Windows:** Chạy nền hoàn toàn ẩn bằng PowerShell
        (`GoogleUnlimited.bat silent`) và tự động tạo lối tắt trong thư
        mục Startup (`GoogleUnlimited.lnk`). Không phụ thuộc vào các
        trình chạy script cũ.
    -   **macOS:** Quản lý dịch vụ LaunchAgent của người dùng bằng
        `launchd`.
    -   **Linux:** Dùng dịch vụ `systemd` cấp người dùng, có hỗ trợ
        phiên chạy nền liên tục (`loginctl enable-linger`).
-   **Bảo vệ chạy một phiên bản duy nhất:**
    -   **Windows:** Mutex hệ thống có tên
        `Local\GoogleUnlimited_Watcher_<path>` ngăn nhiều tiến trình
        theo dõi cùng chạy trên một thư mục cài đặt.
    -   **Unix:** Theo dõi PID riêng bằng `.watcher.pid` và kiểm tra
        tiến trình để ngăn các tiến trình nền xung đột.
-   **Tăng cường bảo mật và quyền riêng tư:**
    -   Tệp thông tin đăng nhập (`gotohp.config`) và cơ sở dữ liệu chỉ
        cấp quyền cho chủ sở hữu (`chmod 600` trên Unix).
    -   Có thể nhập token OAuth trực tiếp, nội dung nhập được ẩn trên
        màn hình để tránh lộ trong lịch sử shell hoặc danh sách tiến
        trình.
    -   Các tệp thực thi trong bản phát hành chính thức được kiểm tra
        bằng mã SHA-256 đáng tin cậy trong quá trình thiết lập.
    -   Kết nối trực tiếp đến máy chủ Google
        (`photoslibrary.googleapis.com`); không có telemetry và không
        thông qua máy chủ trung gian.

------------------------------------------------------------------------

## Yêu cầu

### Yêu cầu theo nền tảng

  ------------------------------------------------------------------------------------------------
  Nền tảng       Phiên bản hệ    Môi trường     Phụ thuộc bắt buộc Cách cài đặt phụ thuộc
                 điều hành /     chạy và shell                     
                 Kiến trúc                                         
  -------------- --------------- -------------- ------------------ -------------------------------
  **Windows**    Windows 10 / 11 PowerShell 5.1 Module `PSSQLite`  Được đóng gói trong
                 (64-bit)        hoặc                              `Modules/PSSQLite/`; tự cài từ
                                 PowerShell 7+                     PowerShell Gallery nếu thiếu
                                 (`pwsh`)                          

  **macOS**      macOS 11 (Big   Bash           `fswatch`,         Tự cài bằng Homebrew khi chạy
                 Sur) trở lên;                  `sqlite3`, `curl`  `./google-unlimited.sh setup`
                 Apple Silicon                                     
                 (M1--M4) và                                       
                 Intel x64                                         

  **Linux**      Các bản phân    Bash           `inotify-tools`,   Tự cài bằng `apt`, `dnf`,
                 phối Linux hiện                `sqlite3`, `curl`  `pacman` hoặc `zypper` khi chạy
                 đại; x86_64                                       `./google-unlimited.sh setup`
                 hoặc                                              
                 aarch64/arm64                                     
  ------------------------------------------------------------------------------------------------

### Các tệp thực thi có sẵn trong kho mã nguồn

-   `gotohp-cli-x64.exe`: Bộ máy tải lên dòng lệnh Windows 64-bit
    (gotohp v0.11.0).
-   `gotohp-x64.exe`: Công cụ giao diện đồ họa trên Windows để đăng nhập
    Google OAuth.
-   `gotohp-cli_amd64`: Tệp thực thi Linux x86_64 gốc.
-   Các tệp thực thi macOS Universal và Linux ARM64 được tự động tải từ
    bản phát hành chính thức và xác minh SHA-256 trong quá trình
    `./google-unlimited.sh setup`.

------------------------------------------------------------------------

## Bắt đầu nhanh

### Bắt đầu nhanh trên Windows

1.  **Sao chép kho mã nguồn:**

    ``` cmd
    git clone https://github.com/shishir0x/GoogleUnlimited.git
    cd GoogleUnlimited
    ```

2.  **Đăng nhập Google Photos:**

    -   Nhấp đúp vào **`gotohp-x64.exe`**.
    -   Chọn **Add Google account** → **Open Google sign-in** → đăng
        nhập → nhấp **I agree**.
    -   Nhấn **F12** trong trình duyệt → tab **Application** (hoặc
        **Storage** trong Firefox) → **Cookies**
        (`https://accounts.google.com`) → sao chép giá trị
        **`oauth_token`**.
    -   Dán token vào `gotohp`, nhấp **Connect account**, sau đó đóng
        cửa sổ.

3.  **Bắt đầu đồng bộ:**

    -   Nhấp đúp **`GoogleUnlimited.bat`** và chọn
        **`[1] Start Live Watcher`**, hoặc chạy:

        ``` cmd
        GoogleUnlimited.bat watch
        ```

### Bắt đầu nhanh trên macOS

1.  **Sao chép kho mã nguồn:**

    ``` bash
    git clone https://github.com/shishir0x/GoogleUnlimited.git
    cd GoogleUnlimited
    ```

2.  **Chạy tập lệnh thiết lập:**

    ``` bash
    chmod +x google-unlimited.sh
    ./google-unlimited.sh setup
    ```

    *(Lệnh này cài `fswatch`, `sqlite3`, tải tệp thực thi Universal và
    xác minh tính toàn vẹn SHA-256.)*

3.  **Đăng nhập Google Photos:**

    -   Mở <https://accounts.google.com/embedded/setup/v2/android> trong
        trình duyệt.

    -   Đăng nhập và nhấp **I agree** *(trang có thể tiếp tục tải mãi,
        đây là hiện tượng bình thường)*.

    -   Nhấn **Option + Command + I** → tab **Application** →
        **Cookies** → sao chép giá trị cookie **`oauth_token`**.

    -   Chạy:

        ``` bash
        ./google-unlimited.sh creds add
        ```

        Dán token khi được yêu cầu (nội dung nhập sẽ được ẩn để bảo
        mật).

4.  **Bắt đầu đồng bộ:**

    ``` bash
    ./google-unlimited.sh watch
    ```

### Bắt đầu nhanh trên Linux

1.  **Sao chép kho mã nguồn:**

    ``` bash
    git clone https://github.com/shishir0x/GoogleUnlimited.git
    cd GoogleUnlimited
    ```

2.  **Chạy tập lệnh thiết lập:**

    ``` bash
    chmod +x google-unlimited.sh
    ./google-unlimited.sh setup
    ```

    *(Lệnh này cài `inotify-tools`, `sqlite3` và `curl` bằng trình quản
    lý gói của hệ thống.)*

3.  **Đăng nhập Google Photos:**

    -   Mở <https://accounts.google.com/embedded/setup/v2/android> trong
        trình duyệt.

    -   Đăng nhập và nhấp **I agree** *(trang tải liên tục là điều được
        dự kiến)*.

    -   Nhấn **F12** → tab **Application** hoặc **Storage** →
        **Cookies** → sao chép giá trị **`oauth_token`**.

    -   Chạy:

        ``` bash
        ./google-unlimited.sh creds add
        ```

        Dán token vào lời nhắc nhập liệu được ẩn. Ngoài ra, có thể
        truyền token qua pipe:

        ``` bash
        echo "<oauth_token>" | ./google-unlimited.sh creds add -
        ```

4.  **Bắt đầu đồng bộ:**

    ``` bash
    ./google-unlimited.sh watch
    ```

------------------------------------------------------------------------

## Xác thực và thiết lập tài khoản

GoogleUnlimited sử dụng quy trình Embedded Setup của Google để xác thực
như một ứng dụng Google Photos trên Android. Cách này cho phép sử dụng
điểm cuối tải lên dành cho thiết bị di động, cần thiết để giả lập Google
Pixel.

### Vì sao trang thiết lập tải mãi không xong?

Khi đăng nhập tại
`https://accounts.google.com/embedded/setup/v2/android` và nhấp **I
agree**, Google chờ một thiết bị Android vật lý xác nhận kết nối. Trong
trình duyệt web, trang sẽ tiếp tục tải hoặc quay vòng vô thời hạn. **Đây
là hiện tượng hoàn toàn bình thường và nằm trong dự kiến.** Cookie xác
thực đã được cấp.

### Lấy `oauth_token`

1.  Mở Công cụ dành cho nhà phát triển (**F12**, hoặc nhấp chuột phải →
    **Inspect**).
2.  Mở công cụ kiểm tra bộ nhớ cookie:
    -   **Chrome, Edge, Brave, Opera:** tab **Application** →
        **Storage** → **Cookies** → `https://accounts.google.com`.
    -   **Firefox:** tab **Storage** → **Cookies** →
        `https://accounts.google.com`.
    -   **Safari:** tab **Storage** → **Cookies** →
        `accounts.google.com`.
3.  Tìm cookie có tên **`oauth_token`** và sao chép toàn bộ giá trị của
    nó.

### Lưu trữ thông tin đăng nhập và bảo mật

-   Thông tin đăng nhập được lưu cục bộ trong `gotohp.config` tại thư
    mục gốc của kho mã nguồn.
-   `gotohp.config` được loại trừ bởi `.gitignore` và **không bao giờ
    được commit** vào hệ thống quản lý phiên bản.
-   Trên macOS và Linux, tập lệnh tự động áp dụng `chmod 600` để chỉ tài
    khoản người dùng hiện tại có thể đọc hoặc ghi tệp.
-   Khi nhập token bằng `./google-unlimited.sh creds add`, nội dung nhập
    trong terminal sẽ bị ẩn để token không xuất hiện trong lịch sử
    terminal, danh sách tiến trình (`ps`) hoặc bị người khác nhìn trộm.

### Quản lý nhiều tài khoản

``` bash
# Liệt kê tất cả tài khoản đã cấu hình (tài khoản đang hoạt động được đánh dấu bằng dấu sao *)
./google-unlimited.sh creds list

# Chuyển tài khoản dùng để tải lên
./google-unlimited.sh creds set user@gmail.com

# Xóa một tài khoản đã cấu hình
./google-unlimited.sh creds remove user@gmail.com
```

*Trên Windows, bạn cũng có thể quản lý tài khoản bằng
`GoogleUnlimited.bat account` hoặc chọn mục `[9]` trong menu.*

------------------------------------------------------------------------

## Tham khảo lệnh

### Windows (`GoogleUnlimited.bat` / PowerShell)

Chạy `GoogleUnlimited.bat` không kèm tham số sẽ mở menu quản lý tương
tác:

``` text
==========================================================
  *  Google Unlimited Photos Auto-Sync (Windows)
==========================================================
  [1] Bắt đầu theo dõi trực tiếp (hiển thị tiến trình)
  [2] Bắt đầu theo dõi nền (chạy ẩn, không có cửa sổ)
  [3] Kiểm tra trạng thái sao lưu
  [4] Xem nhật ký hoạt động
  [5] Bật tự khởi động khi Windows khởi động
  [6] Tắt tự khởi động cùng Windows
  [7] Dừng tiến trình theo dõi nền
  [8] Quản lý thư mục được theo dõi (thêm/liệt kê thư mục tùy chỉnh)
  [9] Quản lý tài khoản Google (liệt kê/chuyển tài khoản)
  [10] Buộc quét lại và đồng bộ toàn bộ phương tiện
  [11] Kiểm tra / cập nhật ứng dụng (giữ nguyên cấu hình an toàn)
  [0] Thoát
==========================================================
```

Bạn cũng có thể chạy lệnh trực tiếp trong Command Prompt hoặc Terminal:

  ---------------------------------------------------------------------------------------------------------------------------------------------------
  Lệnh Batch                              Tham số PowerShell tương ứng                                              Mục menu          Mô tả
  --------------------------------------- ----------------------------------------------------------------- ------------------------- ---------------
  `GoogleUnlimited.bat watch [dirs...]`   `.\WatchAndUpload.ps1 [dirs...]`                                            `[1]`           Bắt đầu theo
                                                                                                                                      dõi trực tiếp ở
                                                                                                                                      chế độ tiền
                                                                                                                                      cảnh, có thanh
                                                                                                                                      tiến trình động
                                                                                                                                      và thời gian dự
                                                                                                                                      kiến hoàn tất
                                                                                                                                      (ETA).

  `GoogleUnlimited.bat silent`            `powershell.exe -WindowStyle Hidden -File .\WatchAndUpload.ps1`             `[2]`           Chạy bộ theo
                                                                                                                                      dõi nền ở chế
                                                                                                                                      độ ẩn, không mở
                                                                                                                                      cửa sổ console.

  `GoogleUnlimited.bat status`            `.\WatchAndUpload.ps1 -StatusOnly`                                          `[3]`           Xem tổng số tệp
                                                                                                                                      đã tải lên, số
                                                                                                                                      lần thử lại
                                                                                                                                      đang chờ, trạng
                                                                                                                                      thái dịch vụ và
                                                                                                                                      10 lượt tải lên
                                                                                                                                      gần nhất.

  `GoogleUnlimited.bat logs`              `.\WatchAndUpload.ps1 -ShowLogs`                                            `[4]`           Hiển thị 30 bản
                                                                                                                                      ghi hoạt động
                                                                                                                                      đồng bộ gần
                                                                                                                                      nhất từ cơ sở
                                                                                                                                      dữ liệu.

  `GoogleUnlimited.bat enable`            `.\WatchAndUpload.ps1 -EnableStartup`                                       `[5]`           Tạo lối tắt
                                                                                                                                      trong thư mục
                                                                                                                                      Startup của
                                                                                                                                      Windows để chạy
                                                                                                                                      khi khởi động
                                                                                                                                      máy.

  `GoogleUnlimited.bat disable`           `.\WatchAndUpload.ps1 -DisableStartup`                                      `[6]`           Xóa lối tắt
                                                                                                                                      khỏi thư mục
                                                                                                                                      Startup của
                                                                                                                                      Windows.

  `GoogleUnlimited.bat stop`              `.\WatchAndUpload.ps1 -StopWatcher`                                         `[7]`           Kết thúc mọi
                                                                                                                                      tiến trình theo
                                                                                                                                      dõi nền đang
                                                                                                                                      hoạt động.

  `GoogleUnlimited.bat dirs`              `.\WatchAndUpload.ps1 -ManageDirs`                                          `[8]`           Mở trình quản
                                                                                                                                      lý thư mục
                                                                                                                                      tương tác hoặc
                                                                                                                                      hộp thoại chọn
                                                                                                                                      thư mục của
                                                                                                                                      Windows.

  `GoogleUnlimited.bat account`           `gotohp-cli creds`                                                          `[9]`           Liệt kê tài
                                                                                                                                      khoản, chuyển
                                                                                                                                      tài khoản đang
                                                                                                                                      hoạt động hoặc
                                                                                                                                      mở giao diện đồ
                                                                                                                                      họa.

  `GoogleUnlimited.bat resync`            `.\WatchAndUpload.ps1 -ForceResync`                                        `[10]`           Quét toàn bộ và
                                                                                                                                      buộc đồng bộ
                                                                                                                                      lại mọi tệp
                                                                                                                                      phương tiện
                                                                                                                                      trong các thư
                                                                                                                                      mục được theo
                                                                                                                                      dõi.

  `GoogleUnlimited.bat update`            `.\WatchAndUpload.ps1 -UpdateApp`                                          `[11]`           Kiểm tra và cài
                                                                                                                                      đặt bản cập
                                                                                                                                      nhật an toàn,
                                                                                                                                      giữ nguyên cấu
                                                                                                                                      hình và cơ sở
                                                                                                                                      dữ liệu.
  ---------------------------------------------------------------------------------------------------------------------------------------------------

### macOS và Linux (`google-unlimited.sh`)

Chạy `./google-unlimited.sh` không kèm tham số trong terminal tương tác
sẽ mở menu quản lý Unix:

``` text
==========================================================
  *  GoogleUnlimited - Unix Manager (Linux/Darwin)
==========================================================
  [1] Bắt đầu theo dõi trực tiếp (tiền cảnh)
  [2] Kiểm tra trạng thái sao lưu
  [3] Xem nhật ký hoạt động
  [4] Chạy thiết lập và cài đặt các thành phần phụ thuộc
  [5] Bật tự khởi động cùng hệ thống (launchd / systemd)
  [6] Tắt tự khởi động cùng hệ thống
  [7] Dừng tiến trình theo dõi nền
  [8] Xác minh tính toàn vẹn của tệp thực thi
  [9] Kiểm tra / cập nhật ứng dụng (giữ nguyên cấu hình an toàn)
  [0] Thoát
==========================================================
```

Bạn có thể thực thi lệnh trực tiếp trong shell:

  -------------------------------------------------------------------------------------------------------
  Lệnh Shell                                               Mục menu             Mô tả
  -------------------------------------------- -------------------------------- -------------------------
  `./google-unlimited.sh watch [dirs...]`                   `[1]`               Bắt đầu theo dõi đồng bộ
                                                                                trực tiếp ở chế độ tiền
                                                                                cảnh.

  `./google-unlimited.sh status`                            `[2]`               Hiển thị thống kê tải
                                                                                lên, số mục trong hàng
                                                                                đợi thử lại và các bản
                                                                                ghi đồng bộ gần đây.

  `./google-unlimited.sh logs`                              `[3]`               Xem 30 bản ghi nhật ký
                                                                                đồng bộ gần nhất, có màu
                                                                                phân biệt mức độ nghiêm
                                                                                trọng.

  `./google-unlimited.sh setup`                             `[4]`               Cài các thành phần phụ
                                                                                thuộc của nền tảng, tải
                                                                                các tệp thực thi còn
                                                                                thiếu và thiết lập quyền
                                                                                truy cập tệp.

  `./google-unlimited.sh enable`                            `[5]`               Bật tự khởi động khi đăng
                                                                                nhập (`launchd` trên
                                                                                macOS, dịch vụ người dùng
                                                                                `systemd` trên Linux).

  `./google-unlimited.sh disable`                           `[6]`               Tắt và gỡ dịch vụ chạy
                                                                                nền tự khởi động.

  `./google-unlimited.sh stop`                              `[7]`               Dừng mọi tiến trình theo
                                                                                dõi nền và bộ lắng nghe
                                                                                hệ thống tập tin đang
                                                                                hoạt động.

  `./google-unlimited.sh verify`                            `[8]`               Xác minh tính toàn vẹn
                                                                                mật mã SHA-256 của tệp
                                                                                thực thi `gotohp-cli`.

  `./google-unlimited.sh update`                            `[9]`               Kiểm tra và cài đặt bản
                                                                                cập nhật an toàn, giữ
                                                                                nguyên cấu hình và cơ sở
                                                                                dữ liệu.

  `./google-unlimited.sh creds [subcommand]`                 ---                Quản lý tài khoản:
                                                                                `creds add`,
                                                                                `creds list`,
                                                                                `creds set <email>`,
                                                                                `creds remove <email>`.

  `./google-unlimited.sh menu`                               ---                Mở menu terminal tương
                                                                                tác.

  `./google-unlimited.sh help`                               ---                Hiển thị thông tin hướng
                                                                                dẫn sử dụng dòng lệnh.
  -------------------------------------------------------------------------------------------------------

------------------------------------------------------------------------

## Giám sát thư mục và định dạng phương tiện được hỗ trợ

### Các thư mục được giám sát

#### Hành vi mặc định (khi không tồn tại `watch_dirs.txt`)

-   **Windows:** Bộ theo dõi tự động giám sát các thư mục ảnh và tải
    xuống tiêu chuẩn:

    -   `Pictures` (thư mục Ảnh của người dùng)
    -   `Downloads` (được xác định thông qua registry hoặc
        `%USERPROFILE%\Downloads`)
    -   Các thư mục đám mây nếu tồn tại trên hệ thống tập tin:
        -   OneDrive: `%USERPROFILE%\OneDrive\Pictures`,
            `%USERPROFILE%\OneDrive\Pictures\Camera Roll`
        -   iCloud: `%USERPROFILE%\Pictures\iCloud Photos\Photos`
        -   Google Drive: `%USERPROFILE%\Google Drive\Google Photos`,
            `%USERPROFILE%\Google Photos`

    *(Lưu ý: Các thư mục Windows khác như Desktop, Videos, Documents và
    Music có thể dễ dàng được thêm bằng lệnh
    `GoogleUnlimited.bat dirs`.)*

-   **macOS và Linux:** Tự động phát hiện và giám sát các thư mục hiện
    có trong danh sách:

    -   `$HOME/Pictures`
    -   `$HOME/Downloads`
    -   `$HOME/Google Photos`
    -   `$HOME/OneDrive/Pictures`
    -   `$HOME/Pictures/iCloud Photos/Photos`

    *(Nếu không có thư mục nào trong danh sách tồn tại, `$HOME/Pictures`
    sẽ được tạo và đưa vào giám sát.)*

### Thư mục theo dõi tùy chỉnh (`watch_dirs.txt`)

Khi có tệp `watch_dirs.txt` trong thư mục gốc của kho mã nguồn, bộ theo
dõi chuyển sang **chế độ tùy chỉnh** và chỉ giám sát các thư mục được
chỉ định trong tệp đó (cộng thêm các thư mục được truyền qua tham số
dòng lệnh).

1.  Tạo hoặc chỉnh sửa `watch_dirs.txt` trong thư mục gốc của kho mã
    nguồn (hoặc dùng `GoogleUnlimited.bat dirs` trên Windows).

2.  Thêm mỗi đường dẫn tuyệt đối trên một dòng. Các dòng bắt đầu bằng
    `#` được xem là chú thích và bị bỏ qua. Biến môi trường sẽ được tự
    động thay thế:

    ``` text
    # Các thư mục phương tiện tiêu chuẩn
    %USERPROFILE%\Pictures
    %USERPROFILE%\Downloads

    # Ổ đĩa ngoài hoặc thẻ SD
    D:\Camera_Imports
    E:\Sony_A7_Media

    # Đường dẫn macOS / Linux
    $HOME/Google Photos
    ${HOME}/Media
    ~/Pictures
    /Volumes/ExternalSSD/Photos
    ```

    Trên macOS và Linux, `$VAR`, `${VAR}` và dấu `~` ở đầu đường dẫn sẽ
    được mở rộng. Các biến không xác định được giữ nguyên và biểu thức
    shell không bao giờ được thực thi.

3.  Bạn cũng có thể chỉ định thư mục trực tiếp khi khởi chạy:

    ``` cmd
    GoogleUnlimited.bat watch "D:\Photos" "E:\Videos"
    ```

    ``` bash
    ./google-unlimited.sh watch /Volumes/ExternalSSD/Photos /Volumes/ExternalSSD/Videos
    ```

### Định dạng phương tiện được hỗ trợ (41 định dạng)

Các tệp được đối chiếu với danh sách phần mở rộng được xác minh (không
phân biệt chữ hoa/chữ thường) và phải có dung lượng tối thiểu 10 KB:

-   **Ảnh và hình ảnh thông thường (14 định dạng):** `.jpg`, `.jpeg`,
    `.png`, `.heic`, `.heif`, `.webp`, `.gif`, `.bmp`, `.tiff`, `.tif`,
    `.svg`, `.ico`, `.jfif`, `.avif`
-   **Định dạng RAW từ máy ảnh (11 định dạng):** `.dng`, `.raw`, `.cr2`,
    `.cr3` (Canon), `.nef` (Nikon), `.arw` (Sony), `.orf` (Olympus),
    `.rw2` (Panasonic), `.pef` (Pentax), `.srw` (Samsung), `.raf`
    (Fujifilm)
-   **Video (16 định dạng):** `.mp4`, `.mov` (Apple QuickTime/iPhone),
    `.mkv`, `.avi`, `.m4v`, `.3gp`, `.3g2`, `.wmv`, `.webm`, `.mpg`,
    `.mpeg`, `.mts`, `.m2ts`, `.ts`, `.flv`, `.vob`

*Các tệp có phần mở rộng tạm thời hoặc chưa hoàn chỉnh (như `.tmp`,
`.crdownload`, `.part`, `.db`, `.config`) và các tệp bên trong
`Modules/` sẽ bị loại trừ hoàn toàn.*

### Kiểm tra trạng thái hoàn tất và phát hiện khóa tệp

Để ngăn việc tải lên tệp bị hỏng hoặc chưa hoàn chỉnh:

1.  Khi phát hiện một tệp phương tiện mới, bộ theo dõi kiểm tra xem tệp
    có còn tăng dung lượng hoặc đang được ứng dụng khác ghi hay không
    (ví dụ: trình duyệt đang tải xuống hoặc phần mềm đang xuất video).
2.  Thuật toán xác định trạng thái sẵn sàng sẽ kiểm tra khả năng ghi độc
    quyền:
    -   **Windows:** Thử mở luồng tệp với khóa chia sẻ độc quyền.
    -   **Unix:** Sử dụng `lsof` hoặc `fuser` để xác minh không có tiến
        trình nào khác đang giữ quyền ghi tệp.
3.  Nếu tệp vẫn bị khóa hoặc đang tăng dung lượng, tệp sẽ được đưa vào
    hàng đợi và kiểm tra lại sau mỗi vài giây cho đến khi quá trình ghi
    hoàn tất.

------------------------------------------------------------------------

## Chạy nền

### Chạy nền trên Windows

-   **Khởi chạy im lặng:** Chạy `GoogleUnlimited.bat silent` hoặc chọn
    mục `[2]`. Lệnh này khởi chạy `WatchAndUpload.ps1` bằng PowerShell
    với `-WindowStyle Hidden`, không tạo cửa sổ console hiển thị.

-   **Tự khởi động khi bật máy:** Chạy `GoogleUnlimited.bat enable` hoặc
    chọn mục `[5]`. Thao tác này tạo lối tắt `GoogleUnlimited.lnk` trong
    thư mục Startup của người dùng:

    ``` text
    %APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup\GoogleUnlimited.lnk
    ```

    Lối tắt trỏ trực tiếp đến PowerShell để chạy `WatchAndUpload.ps1`
    với `-WindowStyle Hidden` trong thư mục kho mã nguồn.

-   **Tắt tự khởi động:** Chạy `GoogleUnlimited.bat disable` hoặc chọn
    mục `[6]` để xóa lối tắt.

-   **Dừng tiến trình nền:** Chạy `GoogleUnlimited.bat stop` hoặc chọn
    mục `[7]`. Lệnh này kết thúc an toàn các tiến trình PowerShell đang
    chạy `WatchAndUpload.ps1`.

### Chạy nền trên macOS

-   **Tự khởi động khi đăng nhập:** Chạy `./google-unlimited.sh enable`
    hoặc chọn mục `[5]`.
    -   Xác minh thông tin đăng nhập Google hợp lệ đã tồn tại.

    -   Tạo tệp cấu hình LaunchAgent của người dùng tại:

        ``` text
        ~/Library/LaunchAgents/com.googleunlimited.watcher.plist
        ```

    -   Nạp dịch vụ bằng `launchctl load -w`. Bộ theo dõi sẽ tự khởi
        động khi bạn đăng nhập macOS và tự khởi động lại nếu thoát bất
        ngờ.
-   **Tắt tự khởi động:** Chạy `./google-unlimited.sh disable` hoặc chọn
    mục `[6]`. Lệnh này gỡ dịch vụ bằng `launchctl unload -w` và xóa tệp
    `.plist`.
-   **Dừng bộ theo dõi đang chạy:** Chạy `./google-unlimited.sh stop`
    hoặc chọn mục `[7]`. Lệnh này dừng LaunchAgent, gửi tín hiệu đến PID
    được ghi trong `.watcher.pid` và dừng các tiến trình `fswatch` đang
    hoạt động.

### Chạy nền trên Linux

-   **Tự khởi động khi bật máy:** Chạy `./google-unlimited.sh enable`
    hoặc chọn mục `[5]`.

    -   Xác minh thông tin đăng nhập Google hợp lệ đã tồn tại.

    -   Tạo tệp dịch vụ `systemd` cấp người dùng tại:

        ``` text
        ~/.config/systemd/user/google-unlimited.service
        ```

    -   Thực thi `systemctl --user daemon-reload`, bật và khởi chạy dịch
        vụ.

    -   Bật phiên đăng nhập duy trì bằng `loginctl enable-linger` (nếu
        có), cho phép dịch vụ đồng bộ nền chạy ngay cả trước khi đăng
        nhập vào giao diện đồ họa.

-   **Kiểm tra trạng thái dịch vụ systemd:**

    ``` bash
    systemctl --user status google-unlimited.service
    ```

-   **Tắt tự khởi động:** Chạy `./google-unlimited.sh disable` hoặc chọn
    mục `[6]`. Lệnh này dừng, vô hiệu hóa dịch vụ và xóa tệp unit.

-   **Dừng bộ theo dõi đang chạy:** Chạy `./google-unlimited.sh stop`
    hoặc chọn mục `[7]`. Lệnh này dừng dịch vụ systemd, kết thúc PID
    trong `.watcher.pid` và dừng mọi tiến trình `inotifywait` nền.

### Bảo vệ chỉ chạy một phiên bản

-   **Windows:** `WatchAndUpload.ps1` sử dụng Mutex Windows có tên
    `Local\GoogleUnlimited_Watcher_<cleanBase>`. Nếu đã có một phiên bản
    bộ theo dõi chạy trong thư mục hiện tại, các lần gọi tiếp theo sẽ
    thoát ngay với mã `0`. `GoogleUnlimited.bat silent` thực hiện kiểm
    tra `-CheckRunning` để bảo đảm không tạo tiến trình trùng lặp.
-   **macOS và Linux:** Tập lệnh theo dõi ghi mã định danh tiến trình
    đang hoạt động vào `.watcher.pid`. Lệnh `stop` kiểm tra
    `.watcher.pid` và danh sách tiến trình (`pgrep`) để kết thúc an toàn
    các bộ theo dõi và bộ lắng nghe đang hoạt động mà không ảnh hưởng
    đến tác vụ hệ thống khác.

------------------------------------------------------------------------

## Lưu trữ dữ liệu, quyền riêng tư và bảo mật

### Cấu trúc cơ sở dữ liệu cục bộ (`gphotos.db`)

Toàn bộ lịch sử đồng bộ được theo dõi cục bộ bằng SQLite ở chế độ WAL
(`Write-Ahead Logging` -- ghi nhật ký trước):

``` sql
-- Theo dõi các tệp phương tiện đã tải lên
CREATE TABLE uploads (
    path         TEXT PRIMARY KEY,
    media_key    TEXT,
    uploaded_at  TEXT NOT NULL DEFAULT (datetime('now','localtime')),
    file_name    TEXT,
    file_size    INTEGER,
    content_hash TEXT
);

-- Theo dõi các tệp đang chờ thử tải lên lại
CREATE TABLE retry_queue (
    path         TEXT PRIMARY KEY,
    attempts     INTEGER NOT NULL DEFAULT 0,
    last_attempt TEXT
);

-- Nhật ký hoạt động (tự động xóa sau 30 ngày)
CREATE TABLE logs (
    id        INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp TEXT NOT NULL DEFAULT (datetime('now','localtime')),
    level     TEXT NOT NULL,
    message   TEXT NOT NULL
);
```

### Quyền riêng tư và đích kết nối mạng

-   **Chỉ kết nối trực tiếp:** Quá trình tải lên và xác thực giao tiếp
    trực tiếp từ máy tính của bạn đến API chính thức của Google
    (`accounts.google.com` và `photoslibrary.googleapis.com`). Không có
    máy chủ telemetry, máy chủ thu thập số liệu hoặc máy chủ chuyển tiếp
    của bên thứ ba.

-   **Thông tin đăng nhập lưu cục bộ:** Token OAuth và các tham số máy
    khách Android chỉ được lưu trong `gotohp.config`.

-   **Bảo vệ bằng Git:** Tệp `.gitignore` của kho mã nguồn bảo vệ dữ
    liệu nhạy cảm:

    ``` gitignore
    *.db
    *.db-wal
    *.db-shm
    gphotos.db
    watch_dirs.txt
    gotohp.config
    *.config
    *.token
    *.log
    ```

    Các bản ghi cơ sở dữ liệu cá nhân, thông tin đăng nhập và danh sách
    thư mục theo dõi cục bộ sẽ không bao giờ được commit lên Git.

------------------------------------------------------------------------

## Khắc phục sự cố

### 1. "ACTION REQUIRED: No Google Account Logged In!" (YÊU CẦU THAO TÁC: Chưa đăng nhập tài khoản Google!)

-   **Nguyên nhân:** Không có thông tin đăng nhập trong `gotohp.config`,
    hoặc chưa có tài khoản nào được đánh dấu là đang hoạt động.
-   **Cách khắc phục:**
    -   **Windows:** Chạy `GoogleUnlimited.bat account` → chọn mục `[2]`
        để mở `gotohp-x64.exe`, đăng nhập và kết nối tài khoản.

    -   **macOS / Linux:** Lấy cookie `oauth_token` bằng Công cụ dành
        cho nhà phát triển của trình duyệt, sau đó chạy:

        ``` bash
        ./google-unlimited.sh creds add
        ```

    -   Xác minh rằng có một tài khoản được hiển thị với dấu sao (`*`):

        ``` bash
        ./google-unlimited.sh creds list
        ```

### 2. "Another instance is already running" (Đã có một phiên bản khác đang chạy) trên Windows

-   **Nguyên nhân:** Một tiến trình theo dõi nền đã chạy và đang sở hữu
    Mutex hệ thống.
-   **Cách khắc phục:**
    -   Chạy `GoogleUnlimited.bat status` để kiểm tra bộ theo dõi nền có
        đang hoạt động hay không.
    -   Để dừng, chạy `GoogleUnlimited.bat stop` (hoặc chọn mục `[7]`).

### 3. Tệp bị khóa hoặc vẫn đang tải xuống

-   **Nguyên nhân:** Tệp phương tiện được phát hiện trong lúc quá trình
    tải xuống, xuất tệp hoặc ghi màn hình vẫn đang diễn ra.
-   **Cách khắc phục:** Không cần thao tác thủ công. GoogleUnlimited tự
    động giữ các tệp bị khóa hoặc đang tăng dung lượng trong hàng đợi
    thử lại và xử lý khi tiến trình ghi đã đóng tệp.

### 4. Lỗi Gatekeeper / quyền truy cập trên macOS

-   **Nguyên nhân:** macOS gắn cờ cách ly (`com.apple.quarantine`) cho
    các tệp thực thi được tải xuống.
-   **Cách khắc phục:**
    -   Chạy `./google-unlimited.sh setup` để tự động gỡ cờ cách ly và
        thiết lập quyền thực thi (`chmod 700`).
    -   Nếu giám sát ổ đĩa ngoài hoặc các thư mục được bảo vệ như
        `Downloads`, hãy bảo đảm terminal có quyền **Full Disk Access**
        trong **System Settings → Privacy & Security** của macOS.

### 5. Hạn chế Execution Policy của PowerShell (Windows)

-   **Nguyên nhân:** Cấu hình PowerShell mặc định của Windows có thể hạn
    chế việc chạy tập lệnh chưa được ký.

-   **Cách khắc phục:** `GoogleUnlimited.bat` tự động truyền tham số
    `-ExecutionPolicy Bypass`. Nếu chạy trực tiếp `WatchAndUpload.ps1`,
    hãy dùng:

    ``` powershell
    powershell.exe -ExecutionPolicy Bypass -File .\WatchAndUpload.ps1
    ```

### 6. Kiểm tra nhật ký chẩn đoán

-   Để kiểm tra các lỗi hoặc cảnh báo đồng bộ gần đây:
    -   **Windows:** `GoogleUnlimited.bat logs` (hoặc mục `[4]` trong
        menu).
    -   **macOS / Linux:** `./google-unlimited.sh logs` (hoặc mục `[3]`
        trong menu).
-   *Lưu ý: Không xóa `gphotos.db` để xử lý các lỗi nhỏ. Xóa
    `gphotos.db` sẽ đặt lại lịch sử tải lên cục bộ và khiến
    GoogleUnlimited đánh giá lại, sau đó có thể tải lên lại toàn bộ
    phương tiện hiện có.*

### 7. Phát video trực tuyến, tua video và phát trên Android TV

-   **Triệu chứng:** Khi phát trực tuyến video đã sao lưu đến trình phát
    bên ngoài trên Android TV (ví dụ Kodi, Nova Video Player, VLC), thao
    tác tua tới/lui không hoạt động hoặc khi tiếp tục xem video đang xem
    dở thì video phát lại từ đầu.
-   **Nguyên nhân:** GoogleUnlimited là một tiến trình nền sao lưu/đồng
    bộ cục bộ, tải phương tiện gốc lên Google Photos. Các điểm cuối tải
    xuống trực tiếp/nguyên bản của Google Photos không hỗ trợ header
    HTTP `Range:` (HTTP 206 Partial Content) để tua đến một đoạn byte
    bất kỳ trong tệp phương tiện gốc. Trình phát web/di động tích hợp
    của Google Photos có thể tua và tiếp tục phát vì máy chủ Google tạo
    các luồng DASH/HLS đã chuyển mã với độ phân giải thấp hơn. Quá trình
    này loại bỏ các luồng âm thanh phụ, âm thanh Dolby/DTS và phụ đề
    nhúng.
-   **Cách khắc phục:**
    -   **Phương án A (Rclone WebDAV với VFS Full Cache):** Chia sẻ kho
        Google Photos trong mạng nội bộ hoặc trên Android TV bằng Rclone
        với `--vfs-cache-mode full`:

        ``` bash
        rclone serve webdav gphotos: --vfs-cache-mode full --vfs-read-chunk-size 32M --vfs-read-chunk-size-limit 2G
        ```

        Rclone tiếp nhận yêu cầu đọc theo phạm vi byte của trình phát
        video và tải các đoạn dữ liệu cần thiết theo yêu cầu, sau đó tạo
        phản hồi HTTP 206 tiêu chuẩn tại máy cục bộ. Nhờ vậy, thao tác
        tua và tiếp tục phát hoạt động liền mạch.

    -   **Phương án B (AList Media Proxy):** Kết nối Google Photos làm
        nhà cung cấp lưu trữ trong
        [AList](https://github.com/alist-org/alist) và bật phát trực
        tuyến WebDAV thông qua proxy cục bộ. AList xử lý bộ đệm theo
        từng đoạn và các yêu cầu phạm vi dữ liệu cho trình phát như Nova
        hoặc Kodi, đồng thời giữ nguyên chất lượng phương tiện gốc và âm
        thanh nhiều luồng.

    -   **Phương án C (Bộ đệm mạng phía trình phát):** Trong Kodi, cấu
        hình `advancedsettings.xml` với bộ đệm đĩa/RAM
        (`<buffermode>1</buffermode>`); hoặc trong Nova Video Player,
        bật **Cache to disk** trong cài đặt trình phát video để hỗ trợ
        tua cục bộ trong các đoạn đã được lưu đệm.

### 8. Cảnh báo diệt virus và nhận diện nhầm ("Virus Detected" / Trình duyệt chặn tải xuống)

-   **Triệu chứng:** Khi tải `GoogleUnlimited-main.zip` từ GitHub bằng
    Chrome hoặc Edge, trình duyệt hiển thị
    `GoogleUnlimited-main.zip Failed - Virus detected`; hoặc Windows
    Defender/phần mềm diệt virus bên thứ ba cảnh báo theo heuristic
    chung (ví dụ `Trojan:Win32/Wacatac` hoặc `Trojan.Gen`).
-   **Nguyên nhân:**
    1.  **Giả lập API thiết bị Pixel:** Tệp thực thi tải lên cốt lõi
        (`gotohp-x64.exe` / `gotohp-cli-x64.exe`) là ứng dụng Go mã
        nguồn mở ([`xob0t/gotohp`](https://github.com/xob0t/gotohp)),
        giao tiếp trực tiếp với các điểm cuối tải lên di động nội bộ của
        Google Photos và giả lập header thiết bị Google Pixel để đủ điều
        kiện sử dụng dung lượng sao lưu không giới hạn. Do tệp chưa được
        ký số và giả mạo thuộc tính phần cứng di động qua HTTPS, các
        công cụ heuristic tổng quát có thể nhận diện nhầm đây là mối đe
        dọa.
    2.  **Tính sạch của tập lệnh:** Các phiên bản cũ sử dụng công cụ hỗ
        trợ VBScript (`StartPhotosWatcher.vbs`) và các lệnh gọi P/Invoke
        kernel32 trong bộ nhớ, vốn kích hoạt cơ chế nhận diện hành vi
        đáng ngờ của trình chạy script. Những thành phần này đã được
        loại bỏ hoàn toàn khỏi mã nguồn và thay bằng cơ chế tự động hóa
        PowerShell gốc, rõ ràng hơn.
-   **Cách xử lý:**
    -   **Phương án A (Khuyến nghị -- Git Clone):** Dùng Git để sao chép
        kho mã nguồn thay vì tải tệp ZIP qua trình duyệt. Git tải trực
        tiếp các tệp mã nguồn, tránh cảnh báo nhầm liên quan đến tệp ZIP
        của trình duyệt:

        ``` cmd
        git clone https://github.com/shishir0x/GoogleUnlimited.git
        cd GoogleUnlimited
        ```

    -   **Phương án B (Bỏ qua chặn tải xuống trên Chrome / Edge):**

        -   Trong Chrome hoặc Edge, nhấn `Ctrl + J` để mở trang Tải
            xuống.
        -   Tìm `GoogleUnlimited-main.zip`, nhấp **Details** (hoặc menu
            **...**) và chọn **Keep anyway** / **Keep dangerous file**.

    -   **Phương án C (Cho phép trong Windows Security / Defender):**

        -   Mở Windows Settings → **Privacy & security** → **Windows
            Security** → **Virus & threat protection**.
        -   Nhấp **Protection history**.
        -   Chọn tệp/lượt tải xuống bị chặn, nhấp **Actions**, rồi chọn
            **Allow on device** hoặc **Restore**.

    -   **Xác minh tính toàn vẹn của tệp thực thi:** Bạn có thể tự xác
        minh các tệp thực thi có khớp checksum của bản phát hành chính
        thức hay không bằng PowerShell:

        ``` powershell
        Get-FileHash gotohp-cli-x64.exe -Algorithm SHA256
        ```

        -   `gotohp-cli-x64.exe`:
            `C272A285A03AEEA02A909F0B74D36F84C4A29C235C8DEE47F7DE75C83F15D2B1`
        -   `gotohp-x64.exe`:
            `5C3F757B9E6B2FFD83934B59274136CF4E67C000265E2EAAA4F34DCA6A821088`
        -   `gotohp-cli_amd64`:
            `945021D5DBC36A96F53121EB4F907ACE39D9457CEA1B43107ED1986441B29030`

------------------------------------------------------------------------

## Giới hạn và tuyên bố miễn trừ trách nhiệm

-   **Tiện ích không chính thức:** GoogleUnlimited là tiện ích mã nguồn
    mở độc lập, **không liên kết, không được duy trì, tài trợ hoặc chứng
    thực bởi Google LLC hay Alphabet Inc.**
-   **Thay đổi dịch vụ và chính sách:** Dự án phụ thuộc vào các điểm
    cuối di động của Google Photos và cơ chế giả lập thiết bị được triển
    khai trong `gotohp`. Google có thể thay đổi API, cơ chế xác thực,
    tiêu chí đủ điều kiện của thiết bị hoặc chính sách lưu trữ bất kỳ
    lúc nào. **Không thể bảo đảm dịch vụ hoạt động liên tục, không gián
    đoạn hoặc duy trì dung lượng không giới hạn và không bị trừ quota vô
    thời hạn.**
-   **Tệp giữ chỗ trên đám mây:** Các tệp chỉ tồn tại dưới dạng tệp giữ
    chỗ trên đám mây (ví dụ OneDrive Files On-Demand hoặc tệp iCloud
    Optimize Storage chưa được tải xuống ổ đĩa cục bộ) không thể được bộ
    theo dõi đọc cho đến khi chúng được tải xuống máy.
-   **Thiết bị lưu trữ tháo rời:** Ổ đĩa ngoài và thẻ SD được thêm vào
    `watch_dirs.txt` sẽ được giám sát khi đang kết nối. Nếu ổ đĩa ngoài
    bị ngắt kết nối, việc giám sát sẽ tiếp tục sau khi ổ được kết nối
    lại và quét lại.
-   **Sao lưu độc lập:** Người dùng được khuyến nghị mạnh mẽ nên duy trì
    các bản sao lưu độc lập, lâu dài cho ảnh quan trọng và phương tiện
    cá nhân.

------------------------------------------------------------------------

## Ghi công và giấy phép

-   **Bộ máy Google Photos:** Được cung cấp bởi
    [`xob0t/gotohp`](https://github.com/xob0t/gotohp) của
    [xob0t](https://github.com/xob0t) (giấy phép GNU General Public
    License phiên bản 3.0).
-   **Bộ máy SQLite trên Windows:** Được cung cấp bởi
    [`PSSQLite`](https://github.com/RamblingCookieMonster/PSSQLite) của
    RamblingCookieMonster (giấy phép Apache License 2.0).
-   **Báo lỗi và đóng góp:** Báo lỗi, đề xuất cải tiến hoặc gửi pull
    request trên GitHub: 👉 **[Trang theo dõi vấn đề của
    GoogleUnlimited](https://github.com/shishir0x/GoogleUnlimited/issues)**
