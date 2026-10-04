# PyUI: quét hết ký tự Unicode có nguy cơ tofu (ô vuông)

Nối tiếp `docs/researchs/pyui-ground.md` (mục 4: thanh % OTA) và
`docs/researchs/pyui-keyboard-font-fallback.md` (bàn phím 4 ký tự
`⇪↑←↵`). Chỉ nghiên cứu, chưa sửa code.

## 1. Cách quét

| Bước | Làm gì |
|---|---|
| 1. Grep literal | Duyệt mọi `.py`/`.json`/`.sh` dưới `App/PyUI/`, in mọi dòng chứa ký tự `U+0080+`, kèm codepoint và ngữ cảnh |
| 2. Grep escape | Tìm `\\uXXXX`, `\\UXXXXXXXX`, `\\xXX`, `chr(0x…)` trong `App/PyUI/main-ui/` — **không có kết quả nào** |
| 3. Lọc render-thật | Loại bỏ ký tự chỉ nằm trong comment/log (không qua `Display.render_text`), giữ lại ký tự đi qua đường vẽ chữ |
| 4. Đo cmap | Parser `struct` thuần stdlib đọc bảng `cmap`: subtable format 4 giải đúng (kiểm tra glyphId != 0 qua `idRangeOffset`, bỏ qua cặp `0xFFFF/0xFFFF`), format 12 lấy nguyên range `start..end`. Kiểm tra membership chính xác từng codepoint, không lấy mẫu |

Đường render duy nhất là `Display.render_text` → `TTF_RenderUTF8_Blended`
với đúng 1 font (`App/PyUI/main-ui/display/display.py:627-643`), nên mọi
ký tự dưới đây tofu hay không chỉ phụ thuộc font của `FontPurpose` đó.

## 2. Kiểm kê: ký tự nào được vẽ lên màn hình

### 2.1. Bàn phím ảo (đã biết, nhắc lại)

| Ký tự | Mã | Vị trí |
|---|---|---|
| `⇪` | U+21EA | `App/PyUI/main-ui/display/on_screen_keyboard.py:16,25,90,151` |
| `←` | U+2190 | `App/PyUI/main-ui/display/on_screen_keyboard.py:16,25,156` |
| `↑` | U+2191 | `App/PyUI/main-ui/display/on_screen_keyboard.py:17,26,91,154` |
| `↵` | U+21B5 | `App/PyUI/main-ui/display/on_screen_keyboard.py:17,26,158` |

### 2.2. Dấu ✓ "đã kết nối" (mới)

| Vị trí | Ngữ cảnh |
|---|---|
| `App/PyUI/main-ui/menus/settings/wifi_menu.py:118` | `value_text="✓"` khi Wi-Fi đã kết nối |
| `App/PyUI/main-ui/menus/settings/bluetooth_menu.py:127` | `primary_text="✓ " + tên thiết bị` khi đã pair |
| `App/PyUI/main-ui/devices/gkd/connman_wifi_menu.py:137` | `value_text="✓"` khi đã kết nối (máy GKD) |

Cả 3 đều đi qua `GridOrListEntry` → `Display.render_text`, tức là
render-thật với font của theme.

### 2.3. Mũi tên Konami trong dialog (mới)

| Vị trí | Ngữ cảnh |
|---|---|
| `App/PyUI/main-ui/menus/settings/modes_menu.py:22` | Chuỗi `"↑↑↓↓←→←→BA,START,SELECT"` trong `UserPrompt.prompt_yes_no` (Game Selection Only Mode) |
| `App/PyUI/main-ui/menus/settings/modes_menu.py:34` | Chuỗi tương tự (Simple Mode) |

Dùng 4 mũi tên `↑` U+2191 `↓` U+2193 `←` U+2190 `→` U+2192. Render-thật
qua dialog `UserPrompt`.

### 2.4. Không phải nguy cơ (đã loại)

| Ký tự | Vì sao loại |
|---|---|
| `→` U+2192 trong `sdl2_audio_player.py:427`, `ffmpeg_image_utils.py:95,105,139,169,234`, `pil_image_utils.py:41`, `activity_log.py:33,54,65`, `box_art_scraper.py:34,54`, `grid_view.py:258` | Chỉ nằm trong comment và log file, không qua `render_text` |
| `–` U+2013 trong `miyoo_trim_game_system_utils.py:164`, `muos_game_system_utils.py:40` | Trong comment |
| `"", "", …, −, ‐, ‑` trong `carousel_view.py:153,156,162,168,169` | Trong comment |
| Pin / Wi-Fi / volume trên top bar | Vẽ bằng **ảnh** (`Theme.get_battery_icon`, `get_wifi_icon`, `get_bluetooth_icon`) + chữ số ASCII + `"%"` — `App/PyUI/main-ui/menus/common/top_bar.py:47-80`. An toàn |
| Thanh % OTA hiện tại | Vẽ ô màu bằng `Display.render_box` qua `progress_bar_geometry` (`App/PyUI/main-ui/utils/progress_bar_geometry.py`, handler ở `App/PyUI/main-ui/utils/realtime_message_network_listener.py:185-204`). Không còn dùng `█`/`·`; 2 ký tự này chỉ còn trong lịch sử `pyui-ground.md` mục 4 |

### 2.5. File ngôn ngữ (`App/PyUI/lang/*.json`)

Quét script chữ viết trong 30 file ngôn ngữ:

| Nhóm | File |
|---|---|
| Latin mở rộng (é, ü, ñ, đ/ơ/ư…) | Azerbaijani, Bosnian, Catalan, French, German, Polish, Portuguese, Romanian, Spanish, Turkish, Vietnamese… |
| Cyrillic | Russian, Ukrainian, Serbian |
| Greek | Greek |
| Kana + CJK | Japanese, Chinese (S), Chinese (T) |
| Hangul | Korean |
| Thai/Lao | Thai, Lao |
| Khmer | Khmer |
| Devanagari | Hindi |

Đáng chú ý: **không file ngôn ngữ nào khai báo `"fonts"`** (grep toàn
bộ `App/PyUI/lang/*.json` không trúng), nên đường override font
theo ngôn ngữ ở `App/PyUI/main-ui/menus/language/language.py:240-257`
hiện không cứu được ngôn ngữ nào — font luôn rơi về font theme
(`App/PyUI/main-ui/themes/theme.py:535-536`) hoặc font dự phòng
(`App/PyUI/main-ui/themes/theme.py:560-574`). Thư mục font theo ngôn
ngữ (`Language.get_fonts_dir`, `language.py:94-98` → `App/PyUI/fonts/`)
cũng không chứa font CJK nào (xem mục 3).

## 3. Đo cmap: font nào thiếu ký tự nào

Phương pháp mục 1 bước 4. Probes gồm: 4 phím bàn phím + `→↓` (Konami)
+ `✓` + `·`/`█` (lịch sử OTA) + `–` + mẫu Latin mở rộng
(`đ ơ ư é ü ñ`) + mẫu Cyrillic/Greek/CJK/Kana/Hangul/Thai/Khmer/
Devanagari/Lao. `Y` = có glyph (glyphId != 0), `.` = thiếu.

Cột theo thứ tự: `21EA 2191 2190 21B5 2192 2193 2713 00B7 2588 2013
0111 01A1 01B0 00E9 00FC 00F1 0411 03B1 3042 AC00 4E2D 0E01 1780 0905
0E81`

| Font | Hàng kết quả |
|---|---|
| `Audiowide-Regular.ttf` | `. . . . . . . Y . Y Y . . Y Y Y . . . . . . . . .` |
| `BeVietnamPro-Regular.ttf` | `. . . . . . . Y . Y Y Y Y Y Y Y . . . . . . . . .` |
| `BeVietnamPro-SemiBold.ttf` | `. . . . . . . Y . Y Y Y Y Y Y Y . . . . . . . . .` |
| `BungeeShade-Regular.ttf` | `. Y Y . Y Y . Y Y Y Y Y Y Y Y Y . . . . . . . . .` |
| `EmilysCandy-Regular.ttf` | `. . . . . . . Y . Y . . . Y Y Y . . . . . . . . .` |
| `Orbitron.ttf` | `. . . . . . . . . Y . . . Y Y Y . . . . . . . . .` |
| `PixelifySans.ttf` | `. . . . . . . Y . Y Y . . Y Y Y Y Y . . . . . . .` |
| `PressStart2P-Regular.ttf` | `. Y Y . Y Y . Y . Y Y . . Y Y Y Y Y . . . . . . .` |
| `Silkscreen-Regular.ttf` | `. . . . . . . Y . Y . . . Y Y Y . . . . . . . . .` |
| `VT323-Regular.ttf` | `. . . . . . . Y . Y Y Y Y Y Y Y . . . . . . . . .` |
| `main-ui/themes/font.ttf` (dự phòng) | `Y Y Y . Y Y Y Y Y Y Y Y Y Y Y Y . . . . . . . . .` |
| `Themes/SPRUCE/nunwen.ttf` (mặc định) | `Y Y Y Y Y Y Y Y Y Y Y Y Y Y Y Y Y Y Y Y Y . . . .` |
| `App/PixelReader/.../DejaVuSans.ttf` | `Y Y Y Y Y Y Y Y Y Y Y Y Y Y Y Y Y Y . . . . . . Y` |
| `App/FileManagement/.../JetBrainsMono-Medium.ttf` | `Y Y Y . Y Y Y Y Y Y Y Y Y Y Y Y Y Y . . . . . . .` |
| `spruce/Font Files/Noto.ttf` | `. . . . . . . Y . Y Y Y Y Y Y Y Y Y . . . . . Y .` |

Đối chiếu với `pyui-keyboard-font-fallback.md` mục 4: nhất quán
(theme `font.ttf` thiếu đúng `↵`; họ pixel thiếu cả 4 phím ngoại trừ
`↑←→↓` ở BungeeShade/PressStart2P). Phát hiện mới:

- `nunwen.ttf` (4.7 MB, 594 nhóm cmap format 12) là font full: phủ cả
  CJK/Kana/Hangul/Greek/Cyrillic — chỉ thiếu Thai/Khmer/Devanagari/Lao.
- `DejaVuSans.ttf` đủ mọi ký tự UI nhưng thiếu CJK/Kana/Hangul.
- Không font nào trong `App/PyUI/fonts/` vẽ được `⇪` hay `↵`.

## 4. Phân loại rủi ro

| Mức | Ký tự | Điều kiện tofu |
|---|---|---|
| Chắc chắn tofu ở theme pixel | `⇪` (bàn phím), `↵` (bàn phím) | Thiếu ở **mọi** font `App/PyUI/fonts/` + cả font dự phòng `themes/font.ttf` |
| Tofu ở đa số theme pixel | `↑←→↓` (bàn phím, Konami), `✓` (wifi/bluetooth) | Chỉ BungeeShade/PressStart2P có mũi tên; **không** font pixel nào có `✓` |
| An toàn thực tế | Pin/Wi-Fi/volume top bar, thanh % OTA | Dùng ảnh + `render_box` + ASCII |
| Tofu theo ngôn ngữ | Toàn bộ UI khi chọn Chinese/Japanese/Korean/Thai/Lao/Khmer/Hindi | Chỉ `nunwen.ttf` (theme mặc định) phủ CJK/Kana/Hangul; mọi theme pixel + DejaVuSans đều thiếu. Không có font CJK nào trong `App/PyUI/fonts/` để fallback |
| Rủi ro tiềm ẩn | `·` U+00B7, `█` U+2588 | `Orbitron.ttf` thiếu cả `·`; nếu code cũ dùng chữ vẽ thanh % quay lại thì tofu |

## 5. Chiến lược fallback chung (không chỉ bàn phím)

Kế thừa phương án A–D của `pyui-keyboard-font-fallback.md` mục 5 (vốn
chỉ cứu bàn phím), mở rộng cho mọi ký tự UI:

### S0. Bảng symbol tập trung + ASCII hóa chỗ nào được (làm ngay)

 gom 7 ký tự render-thật (`⇪ ↑ ← ↵ → ↓ ✓`) vào một map duy nhất
(e.g. `menus/common/symbols.py`), mỗi symbol có bản ASCII dự phòng
(`CAP DEL SHF OK -> <- v ^ X`). Nơi vẽ gọi map thay vì literal rải rác
ở `on_screen_keyboard.py:16-26`, `wifi_menu.py:118`,
`bluetooth_menu.py:127`, `connman_wifi_menu.py:137`,
`modes_menu.py:22,34`.

| Ưu | Nhược | File sửa |
|---|---|---|
| Một chỗ sửa hết tofu UI; dialog/toast/menu sau này chỉ dùng map | Mất icon đẹp ở theme đủ glyph nếu ASCII hóa cứng | 5 file gọi + 1 file map mới |

### S1. Probe lúc nạp font, đổi cả font theo purpose (khuyên dùng cùng S0)

Mở rộng ý tưởng C của doc bàn phím: trong `_load_font`
(`App/PyUI/main-ui/display/display.py:406-423`), với **mọi**
`FontPurpose`, probe tập ký tự của S0 bằng `TTF_GlyphIsProvided`
(đã có sẵn trong SDL_ttf + pysdl2 — xem doc bàn phím mục 3, nguồn
https://wiki.libsdl.org/SDL2_ttf/TTF_GlyphIsProvided); thiếu thì dùng
`Themes/SPRUCE/nunwen.ttf` (đo đủ hết ký tự UI, mục 3) cho purpose đó.

| Ưu | Nhược | File sửa |
|---|---|---|
| Không đụng đường render nóng, không vỡ cache texture `(text, purpose, color)` ở `display.py:50-71`; giữ icon đẹp ở theme đủ glyph | Purpose fallback dùng font khác style theme; chưa cứu tên game/ROM chữ lạ | `display.py:406-423`, `themes/theme.py:524-574` |

### S2. Fallback per-glyph lúc render (sửa gốc, để sau)

Như phương án B doc bàn phím: tách chuỗi, glyph thiếu vẽ bằng
`TTF_RenderGlyph32_Blended` từ font dự phòng rồi ghép surface.
Áp dụng chung thì cứu luôn tên ROM/game chữ lạ và mọi ngôn ngữ.

| Ưu | Nhược | File sửa |
|---|---|---|
| Sửa gốc cho mọi màn hình | Đụng đường render nóng + cache; căn baseline phức tạp | `display.py:627-704`, `display/loaded_font.py:1-6` |

### S3. Font CJK + key `fonts` theo ngôn ngữ (cho nhóm ngôn ngữ châu Á)

Bật lại đường đã có mà đang chết (`language.py:240-257` + `get_fonts_dir`
ở `language.py:94-98`): thêm font CJK vào `App/PyUI/fonts/`, khai báo
`"fonts"` trong `lang/Chinese*.json`, `Japanese.json`, `Korean.json`,
`Thai.json`, `Lao.json`, `Khmer.json`, `Hindi.json`.

| Ưu | Nhược | File sửa |
|---|---|---|
| Cứu cả UI CJK mà không phình font theme | Font CJK nặng (nunwen đã 4.7 MB); tốn RAM máy yếu | 7+ file `lang/*.json` + copy font |

### Đề xuất thứ tự

1. **S0 ngay**: hết tofu UI với ~10 dòng + 1 file map.
2. **S1 tiếp**: giữ icon đẹp ở theme đủ glyph, tự cứu ở theme pixel.
3. **S3** nếu muốn hỗ trợ CJK/Thai/Khmer/Hindi tử tế.
4. **S2** để sau cùng (sửa gốc render).

## 6. Từ điển

| Từ | Nghĩa đơn giản |
|---|---|
| tofu | Ô vuông hiện khi font thiếu chữ |
| cmap | Bảng trong file font: mã chữ → hình vẽ |
| probe | Hỏi font "có vẽ được chữ X không" trước khi dùng |
| per-glyph fallback | Chữ nào thiếu thì lấy font khác vẽ đúng chữ đó |
| per-purpose fallback | Cả nhóm chữ cùng mục đích (bàn phím, danh sách…) đổi sang font khác |
