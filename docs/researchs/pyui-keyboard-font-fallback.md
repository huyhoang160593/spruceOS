# PyUI: bàn phím hiện chữ kiểu gì, vì sao ô vuông, sửa fallback ra sao

Nối tiếp `docs/researchs/pyui-ground.md`. Chỉ nghiên cứu, chưa sửa code.

## 1. Bàn phím vẽ ở đâu

| Chuyện gì | Vị trí |
|---|---|
| Class `OnScreenKeyboard`, lưới phím `normal_keys` / `shifted_keys` 6 hàng x 13 cột | `App/PyUI/main-ui/display/on_screen_keyboard.py:10-28` |
| Mỗi phím vẽ bằng `Display.render_text(..., purpose=FontPurpose.ON_SCREEN_KEYBOARD, ...)` | `App/PyUI/main-ui/display/on_screen_keyboard.py:107-112` |
| Ô nhập chữ phía trên cũng dùng `FontPurpose.ON_SCREEN_KEYBOARD` | `App/PyUI/main-ui/display/on_screen_keyboard.py:55-60`, `App/PyUI/main-ui/display/on_screen_keyboard.py:72-77` |
| `App/PyUI/main-ui/apps/pyui_app.py` KHÔNG phải bàn phím, chỉ là `AppConfig` (label/icon/launch) | `App/PyUI/main-ui/apps/pyui_app.py:1-42` |

4 ký tự dễ thành ô vuông trên phím chức năng:

| Phím | Ký tự | Mã | Dòng code |
|---|---|---|---|
| Caps | `⇪` | U+21EA | `App/PyUI/main-ui/display/on_screen_keyboard.py:16` |
| Shift | `↑` | U+2191 | `App/PyUI/main-ui/display/on_screen_keyboard.py:17` |
| Xóa | `←` | U+2190 | `App/PyUI/main-ui/display/on_screen_keyboard.py:16` |
| Enter | `↵` | U+21B5 | `App/PyUI/main-ui/display/on_screen_keyboard.py:17` |

## 2. Font bàn phím lấy từ đâu

| Bước | Chuyện gì | Vị trí |
|---|---|---|
| 1. Ưu tiên font theo ngôn ngữ | `Theme.get_font` gọi `Language.get_font_for_purpose` trước | `App/PyUI/main-ui/themes/theme.py:524-527`, `App/PyUI/main-ui/menus/language/language.py:240-257` |
| 2. Không có thì lấy font của theme | Bàn phím dùng chung `config["list"]["font"]` ghép với đường dẫn theme | `App/PyUI/main-ui/themes/theme.py:535-536` |
| 3. File không tồn tại thì font dự phòng | Trả về `main-ui/themes/font.ttf` | `App/PyUI/main-ui/themes/theme.py:560-566`, `App/PyUI/main-ui/themes/theme.py:569-574` |
| 4. Mở font một lần cho mọi mục đích | `Display.init_fonts` mở 1 `TTF_Font` cho mỗi `FontPurpose` | `App/PyUI/main-ui/display/display.py:218-222` |
| 5. Vẽ chữ | `Display.render_text` gọi `TTF_RenderUTF8_Blended` với đúng 1 font đó | `App/PyUI/main-ui/display/display.py:627-643` |

Điểm mấu chốt: fallback hiện tại chỉ ở mức **file** (có file hay không), không ở mức **glyph** (font có chứa ký tự hay không). Font mở được là dùng, thiếu ký tự nào cũng kệ.

## 3. Vì sao thiếu glyph thì ra ô vuông (tofu)

Thư viện render là SDL_ttf qua binding pysdl2 (`import sdl2.sdlttf`), không phải pygame, không phải PIL. Xem `App/PyUI/main-ui/display/display.py:13-15`.

| Claim | Nguồn |
|---|---|
| `TTF_RenderUTF8_Blended` nhận đúng **một** `TTF_Font`, không có tham số font dự phòng — API không hề có cơ chế fallback | https://wiki.libsdl.org/SDL2_ttf/TTF_RenderUTF8_Blended (SDL_ttf 2.0.12+) |
| SDL_ttf dựng trên FreeType; hàm tra glyph là `FT_Get_Char_Index`, trả về **0 khi ký tự không có trong font**, và glyph 0 luôn là `.notdef` (ô vuông/thùng rỗng) | http://freetype.org/freetype2/docs/reference/ft2-character_mapping.html ("The glyph index. 0 means 'undefined character code'... value 0 always corresponds to the 'missing glyph'") |
| Muốn hỏi font có glyph không thì dùng `TTF_GlyphIsProvided` (16-bit, từ SDL_ttf 2.0.12) hoặc `TTF_GlyphIsProvided32` (32-bit, từ SDL_ttf 2.0.18) | https://wiki.libsdl.org/SDL2_ttf/TTF_GlyphIsProvided, https://wiki.libsdl.org/SDL2_ttf/TTF_GlyphIsProvided32 |
| pysdl2 đã bind sẵn cả 2 hàm trên (kèm `TTF_RenderGlyph32_Blended` để vẽ từng glyph) | https://raw.githubusercontent.com/py-sdl/py-sdl2/master/sdl2/sdlttf.py (`SDLFunc("TTF_GlyphIsProvided", ...)`, `SDLFunc("TTF_GlyphIsProvided32", ..., added='2.0.18')`) |
| File SDL_ttf đi kèm PyUI đã có `TTF_GlyphIsProvided32` (tức SDL_ttf >= 2.0.18 trên máy) | `App/PyUI/dll/libSDL2_ttf-2.0.so` (kiểm bằng `strings`, thấy `TTF_GlyphIsProvided32`, `TTF_RenderGlyph32_Blended`) |
| 4 ký tự bàn phím đều < U+FFFF nên bản 16-bit (`TTF_GlyphIsProvided`, có từ 2.0.12) là đủ, không bắt buộc 2.0.18 | Suy ra từ bảng mã mục 1 + https://wiki.libsdl.org/SDL2_ttf/TTF_GlyphIsProvided |

Đây là cùng một họ lỗi với thanh % OTA đã ghi trong `pyui-ground.md` mục 4 (`█` U+2588, `·` U+00B7 thiếu trong font thì ra ô vuông).

## 4. Đo thực tế: font nào thiếu ký tự nào

Cách đo: đọc bảng `cmap` trong file `.ttf` bằng parser `struct` thuần stdlib (không có fontTools/PIL trên máy dev), kiểm tra codepoint có mặt không. Kết quả:

| Font | `⇪` | `↑` | `←` | `↵` | `█` | `·` |
|---|---|---|---|---|---|---|
| `Themes/SPRUCE/nunwen.ttf` (theme mặc định) | có | có | có | có | có | có |
| `App/PyUI/main-ui/themes/font.ttf` (font dự phòng) | có | có | có | **không** | có | có |
| `App/PixelReader/resources/fonts/DejaVuSans.ttf` | có | có | có | có | có | có |
| `App/PyUI/fonts/PressStart2P-Regular.ttf` | không | có | có | không | không | có |
| `App/PyUI/fonts/BungeeShade-Regular.ttf` | không | có | có | không | có | có |
| `App/PyUI/fonts/Audiowide-Regular.ttf` | không | không | không | không | không | có |
| `App/PyUI/fonts/Orbitron.ttf` | không | không | không | không | không | không |
| `App/PyUI/fonts/VT323-Regular.ttf`, `Silkscreen`, `PixelifySans`, `EmilysCandy`, `BeVietnamPro-*` | không | không | không | không | không | có |

Đọc ra: theme mặc định (SPRUCE + `nunwen.ttf`) đủ glyph nên không lỗi. Theme khác chỉ định font trang trí (kiểu pixel như họ `App/PyUI/fonts/`) thì phím `⇪ ↑ ← ↵` thành ô vuông. Tệ hơn: ngay cả font dự phòng `themes/font.ttf` cũng thiếu `↵` U+21B5, nên đường fallback hiện tại không cứu được phím Enter.

## 5. Phương án fallback

### A. Đổi ký tự lạ thành chữ ASCII an toàn (dễ nhất)

Đổi trong lưới phím: `⇪`→`"CAP"`, `↑`→`"SHF"`, `←`→`"DEL"`, `↵`→`"OK"` (giữ logic so sánh chuỗi ở `App/PyUI/main-ui/display/on_screen_keyboard.py:151-158` đồng bộ).

| Ưu | Nhược | Độ phức tạp | File sửa |
|---|---|---|---|
| Hết tofu 100%, không đụng render/font | Mất biểu tượng đẹp, phím 3 chữ chật hơn trên màn 640px | Rất thấp (~10 dòng) | `App/PyUI/main-ui/display/on_screen_keyboard.py:12-28` |

### B. Vẽ từng glyph thiếu bằng font dự phòng (đúng nghĩa fallback)

Trong `Display.render_text`: tách chuỗi thành từng ký tự, hỏi `sdl2.sdlttf.TTF_GlyphIsProvided(font, ord(ch))`; ký tự thiếu thì vẽ bằng `TTF_RenderGlyph32_Blended` từ font dự phòng rồi ghép surface lại trước khi tạo texture.

| Ưu | Nhược | Độ phức tạp | File sửa |
|---|---|---|---|
| Giữ nguyên look theme, sửa gốc cho mọi màn hình (cả thanh % OTA), API đã có sẵn trong SDL_ttf + pysdl2 | Phải sửa đường render nóng + cache texture (key hiện tại là `(text, purpose, color)` ở `App/PyUI/main-ui/display/display.py:50-71`, phải tính fallback vào key); căn lề/ghép glyph khác baseline dễ lệch | Trung bình-cao | `App/PyUI/main-ui/display/display.py:627-704`, `App/PyUI/main-ui/display/loaded_font.py:1-6`, `App/PyUI/main-ui/themes/theme.py:569-574` |

### C. Kiểm tra 1 lần lúc nạp font: thiếu glyph bàn phím thì đổi cả font (khuyên dùng cùng A)

Lúc `_load_font` (`App/PyUI/main-ui/display/display.py:406-423`), nếu `purpose == ON_SCREEN_KEYBOARD` thì gọi `TTF_GlyphIsProvided` cho 4 ký tự mục 1; thiếu ký tự nào thì mở `DejaVuSans.ttf` (đã có sẵn trong repo, đo đủ cả 6 ký tự, mục 4) thay cho font theme, chỉ cho mục đích bàn phím.

| Ưu | Nhược | Độ phức tạp | File sửa |
|---|---|---|---|
| Không đụng đường render nóng, không vỡ cache, cả bàn phím đồng nhất 1 font | Toàn bàn phím dùng font khác theme khi fallback (lệch style); chỉ cứu bàn phím, không cứu chỗ khác | Thấp | `App/PyUI/main-ui/display/display.py:406-423`, `App/PyUI/main-ui/themes/theme.py:524-574` (+ copy `DejaVuSans.ttf` vào chỗ PyUI đọc được) |

### D. Thêm key config theme `keyboardFont` / `fontFallback` + patch mặc định

Theme được chỉ định riêng font bàn phím; `ThemePatcher` tự điền `DejaVuSans.ttf` khi theme thiếu key. Đi kèm UI chọn font (hiện tại `theme_settings_fonts.py` còn **loại trừ** `ON_SCREEN_KEYBOARD` khỏi chỉnh size — `App/PyUI/main-ui/menus/settings/theme/theme_settings_fonts.py:21`, và `set_font_size` bỏ qua case này ở `App/PyUI/main-ui/themes/theme.py:642-643`).

| Ưu | Nhược | Độ phức tạp | File sửa |
|---|---|---|---|
| Linh hoạt cho theme maker, không hardcode | Phải định nghĩa schema config + migrator + UI; theme cũ không có key vẫn lỗi nếu không kèm C | Trung bình | `App/PyUI/main-ui/themes/theme.py:524-574`, `App/PyUI/main-ui/themes/theme_patcher.py`, `App/PyUI/main-ui/menus/settings/theme/theme_settings_fonts.py:10-30` |

## 6. Đề xuất thứ tự

1. **A ngay**: hết tofu cho mọi theme, không rủi ro.
2. **C tiếp**: giữ được biểu tượng `⇪ ↑ ← ↵` ở theme đủ glyph, tự cứu ở theme thiếu glyph.
3. **B** để sau nếu muốn sửa gốc cho cả thanh % OTA và tên game/ROM lạ chữ.
4. **D** chỉ khi muốn mở cho theme maker tự chọn.

## 7. Từ điển

| Từ | Nghĩa đơn giản |
|---|---|
| cmap | Bảng trong file font: mã chữ → hình vẽ. Không có mã là không vẽ được. |
| .notdef / tofu | Hình vẽ số 0 trong font, hiện khi ký tự không có. Nhìn như ô vuông. |
| SDL_ttf | Thư viện vẽ chữ PyUI đang dùng (qua pysdl2), mỗi lần vẽ chỉ dùng 1 font. |
| TTF_GlyphIsProvided | Hàm hỏi: "font này có vẽ được ký tự X không?". |
| font.ttf dự phòng | `App/PyUI/main-ui/themes/font.ttf`. Dùng khi theme mất file font, nhưng bản thân nó thiếu `↵`. |
