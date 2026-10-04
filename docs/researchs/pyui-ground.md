# PyUI: cách nó chạy (dễ hiểu)

PyUI là menu viết bằng Python. Chạy trên máy chơi game cầm tay.

## 1. Khởi động: 5 bước

| Bước | Chuyện gì xảy ra | Vị trí |
|---|---|---|
| 1. Vào | File `launch.sh` chọn máy. Rồi gọi `mainui.py` kèm tên máy. | `App/PyUI/launch.sh:34-39` |
| 2. Đọc cấu hình | Đọc file cấu hình. Đọc ngôn ngữ. | `App/PyUI/main-ui/mainui.py:242-247` |
| 3. Nạp giao diện | Chọn thư mục theme. Gọi `Theme.init`. | `App/PyUI/main-ui/mainui.py:264-269` |
| 4. Bật màn hình | Gọi `Display.init`. Gọi `Display.present` để hiện khung đầu. | `App/PyUI/main-ui/mainui.py:271-274` |
| 5. Nhận nút | Gọi `Controller.init`. Rồi vào vòng lặp `MainMenu` đợi nút bấm. | `App/PyUI/main-ui/mainui.py:278`, `App/PyUI/main-ui/mainui.py:287-298` |

`Controller.init` chỉ lấy bộ điều khiển của máy đang dùng. Xem `App/PyUI/main-ui/controller/controller.py:81-83`.

## 2. Cách vẽ màn hình

Đơn giản: vẽ nháp trước. Hiện một lần sau.

| Ý | Giải thích | Vị trí |
|---|---|---|
| Vẽ nháp | `Display.init` tạo một tấm vẽ ẩn tên là `render_canvas`. Màn nào cũng vẽ lên tấm này trước. | `App/PyUI/main-ui/display/display.py:118-129` |
| Hiện ra | `Display.present` đem tấm vẽ ẩn dán lên màn hình thật. Có xoay và phóng to nếu máy cần. | `App/PyUI/main-ui/display/display.py:1020-1048` |
| Chọn giao diện | Có file riêng theo độ phân giải thì dùng. Ví dụ `config_640x480.json`. Không có thì dùng `config.json`. | `App/PyUI/main-ui/themes/theme.py:39-49` |
| Phóng chữ | Máy to hơn 640x480 thì nhân kích thước chữ và ô. Gốc so với 640x480. | `App/PyUI/main-ui/themes/theme.py:72-86` |

```
[View] --vẽ--> [render_canvas (tấm ẩn)] --present--> [màn hình thật]
```

## 3. Thanh % khi tải update (OTA)

Đường đi từ file tải xuống tới mắt người xem.

| Bước | Chuyện gì xảy ra | Vị trí |
|---|---|---|
| 1. Gọi tải | `downloader.sh` gọi hàm `download_and_display_progress`. | `App/-OTA/downloader.sh:627` |
| 2. Vòng lặp | Hàm này tải file ở nền. Mỗi 0.1 giây đo file được bao nhiêu. Tính ra %. Gọi hàm hiển thị. | `spruce/scripts/helperFunctions.sh:982-1029` |
| 3. Gửi tin | Hàm hiển thị gửi chuỗi JSON `{"cmd":"TEXT_WITH_PERCENTAGE_BAR"}` qua socket. | `spruce/scripts/helperFunctions.sh:922-932`, `spruce/scripts/helperFunctions.sh:879-895` |
| 4. Vẽ thanh | PyUI nhận JSON. Xóa màn hình. Vẽ chữ. Vẽ thanh. Vẽ dòng MB. Rồi hiện. | `App/PyUI/main-ui/utils/realtime_message_network_listener.py:185-204` |
| 5. Cách chia | Thanh có 20 ô. Mỗi ô là 5%. Làm tròn % về bội của 5. | `App/PyUI/main-ui/utils/realtime_message_network_listener.py:117-123` |

Ví dụ 47% làm tròn về 45%. Số ô đặc là 9. Chuỗi ra là `[█████████···········] 47%`.

## 4. Vì sao font làm hỏng thanh %

Thanh dùng 2 ký tự đặc biệt để vẽ.

| Ký tự | Mã | Nghĩa |
|---|---|---|
| `█` | U+2588 | Ô đặc (phần đã tải). |
| `·` | U+00B7 | Ô mờ (phần chưa tải). |

Vấn đề:

| Lỗi | Vì sao | Vị trí |
|---|---|---|
| Ô vuông | Font đang dùng không có 2 ký tự này. Máy vẽ ô trống hoặc trắng. | `App/PyUI/main-ui/display/display.py:218-222` |
| Nhảy chiều rộng | Font thường mỗi chữ rộng khác nhau. Dấu `[` và `]` không thẳng hàng. Thanh dài ra thụt vào khi % tăng. | `App/PyUI/main-ui/display/display.py:627-704` |
| Dấu `%` lạc | Dấu `[`, `]`, `%` mỗi font vẽ một kiểu. Nhìn khác nhau tùy theme. | `App/PyUI/main-ui/themes/theme.py:524-630` |

Hướng sửa:

| Cách | Ý tưởng |
|---|---|
| Vẽ hộp | Dùng `render_box` vẽ ô màu thay vì dùng chữ. Nên dùng cách này. Xem `App/PyUI/main-ui/display/display.py:814-817`. |
| Font riêng | Dùng một font cố định chiều rộng chỉ cho thanh %. |
| Dùng ảnh | Dùng 2 ảnh có sẵn: khung và phần đặc. Xem `App/-OTA/imgs/downloadBar.png`, `App/-OTA/imgs/downloadFill.png`. |

## 5. Từ điển

| Từ | Nghĩa đơn giản |
|---|---|
| Theme | Bộ quần áo: màu, chữ, ảnh nền. |
| Display | Thợ vẽ: giữ tấm vẽ ẩn và dán lên màn hình. |
| render_canvas | Tấm vẽ ẩn. Vẽ nháp ở đây rồi mới hiện. |
| ViewType | Kiểu màn hình: lưới, danh sách, ảnh kèm chữ. |
| TEXT_WITH_PERCENTAGE_BAR | Lệnh: "hiện chữ kèm thanh %". Đi qua socket. |
| progress_bar | Hàm làm chuỗi `[███···] %` từ con số %. |
