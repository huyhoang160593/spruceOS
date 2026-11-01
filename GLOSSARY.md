# spruceOS PyUI

Giao diện menu PyUI trên máy chơi game cầm tay (OTA update, updater, theme theo máy).

## Language

**Ô fill**:
Ô vuông đã tải trong thanh tiến trình, màu nhấn của theme.
_Avoid_: ô sáng, ô đặc, ô trắng

**Ô track**:
Ô vuông chưa tải trong thanh tiến trình, cùng hue với ô fill nhưng pha tối để luôn phân biệt được.
_Avoid_: ô nền, ô mờ, ô xám

**font fallback**:
Cơ chế vẽ ký tự mà font theme thiếu bằng font dự phòng cùng cỡ, để không còn ô vuông.
_Avoid_: font thay thế, font dự phòng sai cỡ

**tofu**:
Ô vuông □ hiện thay cho ký tự khi font đang dùng thiếu glyph đó.
_Avoid_: ô lỗi, ký tự rác

**run**:
Đoạn ký tự liên tiếp trong một chuỗi được vẽ bằng cùng một font (primary hoặc fallback).
_Avoid_: đoạn chữ, chunk

**symbol font**:
Font giao diện dùng chủ đích cho glyph nút và phím, chứ không phải để lấp chỗ font theme còn thiếu.
_Avoid_: font icon, font thay thế

**symbol run**:
Đoạn ký tự được vẽ bằng symbol font, trong khi phần còn lại của cùng dòng vẫn theo font purpose của dòng đó.
_Avoid_: đoạn glyph, ký tự đặc biệt
