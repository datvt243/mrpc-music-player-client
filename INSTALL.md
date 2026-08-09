# Hướng dẫn cài đặt

Chọn hệ điều hành: [macOS](#macos) · [Windows](#windows)

---

# macOS

## Yêu cầu

- macOS
- [Homebrew](https://brew.sh) đã cài
- Thư mục chứa nhạc (mp3/flac/m4a/...) ở một nơi **thường**, không nằm trong thư viện Apple Music (`~/Music/Music/Media.localized`)

## Bước 1 — Cài các gói qua Homebrew

```bash
brew install mpd mpc rmpc
```

- `mpd` — daemon phát nhạc
- `mpc` — CLI test nhanh
- `rmpc` — TUI client chính

## Bước 2 — Clone repo này vào đúng vị trí config của mpd

```bash
git clone <URL-repo-cua-ban> ~/.config/mpd
```

> Nếu `~/.config/mpd` đã có sẵn nội dung (do brew tự tạo lúc install), backup rồi merge lại, đừng ghi đè mất `database`/`state` nếu bạn đã có sẵn thư viện quét trước đó.

## Bước 3 — Đặt config của rmpc

```bash
mkdir -p ~/.config/rmpc
cp ~/.config/mpd/rmpc/config.ron ~/.config/rmpc/config.ron
```

## Bước 4 — Sửa `mpd.conf` cho đúng máy của bạn

Mở `~/.config/mpd/mpd.conf`, sửa 2 chỗ:

1. **Username** — repo này dùng đường dẫn tuyệt đối kiểu `/Users/USERNAME/...`. Thay `USERNAME` bằng username thật của bạn (hoặc chạy lệnh dưới để tự thay bằng `$HOME` hiện tại):

   ```bash
   sed -i '' "s#/Users/[^/]*#$HOME#g" ~/.config/mpd/mpd.conf
   ```

2. **`music_directory`** — trỏ đúng vào thư mục chứa nhạc thật của bạn, ví dụ:

   ```
   music_directory "/Users/ban/Workspace/Music"
   ```

   ⚠️ **Không trỏ vào `~/Music/Music/Media.localized`** (thư viện Apple Music) — macOS chặn quyền đọc file ở đó (lỗi `Operation not permitted`), mpd sẽ không decode được nhạc dù thấy file trong danh sách.

File `mpd.conf` mẫu trông như sau:

```ini
music_directory "/Users/ban/Workspace/Music"
db_file "/Users/ban/.config/mpd/database"
log_file "/Users/ban/.config/mpd/log"
pid_file "/Users/ban/.config/mpd/pid"
state_file "/Users/ban/.config/mpd/state"
restore_paused "yes"
playlist_directory "/Users/ban/.config/mpd/playlists"
bind_to_address "127.0.0.1"
port "6600"
audio_output {
    type "osx"
    name "CoreAudio"
    mixer_type "software"
}
```

## Bước 5 — Tạo symlink để mpd tìm được config kể cả khi chạy nền qua launchd

Đây là bước quan trọng, đừng bỏ qua (giải thích ở phần [Troubleshooting](#troubleshooting--các-lỗi-đã-gặp) bên dưới):

```bash
ln -sf ~/.config/mpd/mpd.conf /opt/homebrew/etc/mpd.conf
```

> Máy Intel Mac dùng Homebrew ở `/usr/local` thay vì `/opt/homebrew` — đổi đường dẫn tương ứng: `/usr/local/etc/mpd.conf`.

## Bước 6 — Khởi động mpd như service nền

```bash
brew services start mpd
```

Kiểm tra service chạy ổn định (không bị crash-loop):

```bash
brew services list | grep mpd
# phải thấy trạng thái "started", không phải "error"
```

## Bước 7 — Quét thư viện nhạc

```bash
mpc update
mpc stats
```

`mpc stats` phải hiện số Artists/Albums/Songs > 0.

## Bước 8 — Test phát nhạc bằng mpc

```bash
mpc add /
mpc play
sleep 2
mpc status
```

Nếu thấy thời gian phát tăng dần (`0:00` → `0:0x`) và không có dòng lỗi nào trong `~/.config/mpd/log`, tức là âm thanh đã hoạt động.

```bash
mpc stop
mpc clear
```

## Bước 9 — Mở rmpc

```bash
rmpc
```

rmpc sẽ tự kết nối tới `127.0.0.1:6600` (theo `~/.config/rmpc/config.ron`) và hiển thị thư viện vừa quét.

---

## Troubleshooting (macOS) — các lỗi đã gặp

### 1. Không có tiếng, log báo `Failed to open "CoreAudio" (osx)` hoặc dùng nhầm `type "pulse"`

macOS không có PulseAudio. `audio_output` trong `mpd.conf` phải dùng:

```
audio_output {
    type "osx"
    name "CoreAudio"
    mixer_type "software"
}
```

### 2. `brew services list` báo trạng thái `error`, mpd cứ restart liên tục

Nguyên nhân: mpd chạy qua `launchd` (brew services) **không có biến môi trường `HOME`**, nên không tự tìm được `~/.config/mpd/mpd.conf` (dấu `~` không expand được) → mpd thoát ngay lúc khởi động → launchd restart lặp vô hạn.

Cách khắc phục bền vững (đã áp dụng trong repo này):

- Dùng đường dẫn **tuyệt đối** trong `mpd.conf` thay vì `~`.
- Symlink config vào `/opt/homebrew/etc/mpd.conf` — đây là một trong các đường dẫn mặc định mpd tự tìm **không phụ thuộc** biến `HOME`, nên hoạt động cả khi chạy qua launchd.

Không nên sửa trực tiếp file plist ở `~/Library/LaunchAgents/homebrew.mxcl.mpd.plist` để thêm `HOME` — file này sẽ bị Homebrew ghi đè mỗi khi chạy `brew services start/restart` hoặc `brew upgrade mpd`.

Nếu sau này `brew upgrade mpd` làm mất symlink, chỉ cần tạo lại:

```bash
ln -sf ~/.config/mpd/mpd.conf /opt/homebrew/etc/mpd.conf
brew services restart mpd
```

### 3. mpd thấy file trong danh sách nhưng phát báo lỗi `Failed to decode ... Operation not permitted`

Do `music_directory` trỏ vào thư viện Apple Music (`~/Music/Music/Media.localized`) — macOS bảo vệ (TCC) thư mục này, chỉ ứng dụng Music/Photos mới đọc được. Chuyển nhạc ra một thư mục thường (ví dụ `~/Workspace/Music`) và cập nhật lại `music_directory`, sau đó `mpc update`.

### 4. Sau khi đổi `music_directory`, lần đầu update báo 1 dòng lỗi `Failed to decode ... No such file or directory`

Vô hại — đây là do `state_file` cũ lưu bài đang phát dở của thư viện nhạc trước đó, mpd cố resume nhưng đường dẫn cũ không còn khớp thư mục mới. Chỉ cần:

```bash
mpc stop
mpc clear
```

rồi phát lại bình thường.

---

# Windows

> ⚠️ **Lưu ý:** rmpc [chỉ chính thức hỗ trợ Linux](https://rmpc.mierak.dev) — Windows không nằm trong danh sách được test/support chính thức. Phần dưới đây tổng hợp từ tài liệu chính thức của mpd/rmpc, **chưa được test trực tiếp** trên máy Windows thật (khác với phần macOS ở trên đã test live). Nếu gặp lỗi phát sinh không có trong Troubleshooting, tạo issue trên repo này kèm log để cập nhật thêm.

Có 2 cách, chọn 1:

- **Cách 1 — WSL2 (khuyên dùng):** cài mpd/mpc/rmpc y hệt Linux bên trong WSL2, ổn định, đúng theo hỗ trợ chính thức của rmpc. Nhược điểm: cần cấu hình audio bridge từ WSL ra loa Windows.
- **Cách 2 — Windows gốc (native, best-effort):** dùng bản `mpd.exe` chính chủ từ musicpd.org + build rmpc bằng Cargo. Đơn giản hơn về audio (dùng thẳng WASAPI/WinMM) nhưng không có gì đảm bảo mọi tính năng của rmpc (đặc biệt album art qua terminal image protocol) hoạt động đúng.

## Cách 1 — WSL2 (khuyên dùng)

### Bước 1 — Cài WSL2 + Ubuntu

Mở PowerShell (Run as Administrator):

```powershell
wsl --install -d Ubuntu
```

Khởi động lại máy nếu được yêu cầu, sau đó mở Ubuntu từ Start Menu để tạo user Linux.

### Bước 2 — Trong Ubuntu (WSL), cài mpd/mpc

```bash
sudo apt update
sudo apt install -y mpd mpc git
```

### Bước 3 — Clone repo và setup config (giống hệt Linux/macOS)

```bash
git clone https://github.com/datvt243/mrpc-music-player-client.git ~/.config/mpd
sed -i "s#/Users/[^/]*#$HOME#g" ~/.config/mpd/mpd.conf
```

Sửa `music_directory` trong `~/.config/mpd/mpd.conf` trỏ vào thư mục nhạc thật (ví dụ nhạc để trong Windows ở `D:\Music` thì trong WSL sẽ là `/mnt/d/Music`), và đổi `audio_output` sang:

```
audio_output {
    type "pulse"
    name "PulseAudio"
}
```

### Bước 4 — Bridge audio từ WSL ra Windows

WSLg (đi kèm sẵn trên Windows 11 / WSL bản mới) đã có sẵn PulseAudio server chuyển tiếp âm thanh ra loa Windows — thường không cần cấu hình thêm. Kiểm tra nhanh:

```bash
speaker-test -c 2 -t sine -l 1 2>&1 | head -5
```

Nếu không nghe được gì, xem thêm hướng dẫn chính thức: [WSL audio support](https://learn.microsoft.com/en-us/windows/wsl/tutorials/gui-apps).

### Bước 5 — Chạy mpd, cài rmpc, test

```bash
mkdir -p ~/.config/rmpc && cp ~/.config/mpd/rmpc/config.ron ~/.config/rmpc/config.ron
mpd ~/.config/mpd/mpd.conf
mpc update && mpc stats
mpc add / && mpc play && sleep 2 && mpc status   # test có tiến trình phát không có lỗi
mpc stop && mpc clear

curl https://sh.rustup.rs -sSf | sh    # nếu chưa có Rust
cargo install rmpc
rmpc
```

## Cách 2 — Windows gốc (native, best-effort)

### Bước 1 — Tải mpd.exe chính thức

Tải bản binary chính thức (không cần cài đặt, chỉ là 1 file `.exe`):

```
https://www.musicpd.org/download/win32/0.24.13/mpd.exe
```

(kiểm tra bản mới nhất tại https://www.musicpd.org/download/win32/). Tạo thư mục `C:\mpd\` và bỏ `mpd.exe` vào đó.

### Bước 2 — Tạo config `C:\mpd\mpd.conf`

Copy nội dung `mpd.conf` trong repo này, sửa đường dẫn sang kiểu Windows và đổi `audio_output` sang `wasapi`:

```ini
music_directory "C:/Users/ban/Music"
db_file "C:/mpd/database"
log_file "C:/mpd/log"
pid_file "C:/mpd/pid"
state_file "C:/mpd/state"
restore_paused "yes"
playlist_directory "C:/mpd/playlists"
bind_to_address "127.0.0.1"
port "6600"
audio_output {
    type "wasapi"
    name "WASAPI"
}
```

> Dùng dấu `/` thay vì `\` trong đường dẫn (mpd đọc dạng Unix-style path, kể cả trên Windows) — không dùng ổ đĩa mạng hoặc thư mục cần quyền đặc biệt (OneDrive-protected folder có thể gây lỗi đọc file tương tự lỗi TCC trên macOS).

Nếu `wasapi` không hoạt động (một số bản build cũ không có backend này), đổi sang `winmm`:

```ini
audio_output {
    type "winmm"
    name "WinMM"
}
```

### Bước 3 — Chạy thử (foreground) để kiểm tra config đúng

Mở PowerShell tại `C:\mpd\`:

```powershell
.\mpd.exe --no-daemon --stdout C:\mpd\mpd.conf
```

Không có lỗi hiện ra tức là ổn, `Ctrl+C` để dừng.

### Bước 4 — Cho mpd tự chạy nền

Cách đơn giản nhất (không cần quyền admin): tạo shortcut chạy `mpd.exe C:\mpd\mpd.conf` và bỏ vào thư mục Startup (`Win + R` → gõ `shell:startup`), hoặc tạo task trong **Task Scheduler** với trigger "At log on".

Muốn chạy như Windows Service thật sự (khởi động cả khi chưa đăng nhập) thì dùng [NSSM](https://nssm.cc/) để wrap `mpd.exe` — `sc create` trực tiếp thường không hoạt động vì mpd.exe không tự implement Service Control protocol của Windows.

### Bước 5 — Cài rmpc qua Cargo

rmpc chưa có bản build sẵn cho Windows, cần build từ source bằng Rust:

```powershell
# Cài Rust nếu chưa có: https://rustup.rs
cargo install rmpc
```

Copy config rmpc trong repo (`rmpc/config.ron`) ra một chỗ trên máy Windows, ví dụ `C:\Users\ban\.rmpc\config.ron`, rồi chạy chỉ định thẳng bằng flag `--config` (an toàn hơn là đoán default path trên Windows):

```powershell
rmpc --config "C:\Users\ban\.rmpc\config.ron"
```

### Bước 6 — Test bằng `mpc` (tuỳ chọn)

`mpc` không có bản build sẵn cho Windows trên musicpd.org. Có thể bỏ qua bước test bằng `mpc` và test trực tiếp bằng rmpc, hoặc tự build `mpc` từ [source](https://github.com/MusicPlayerDaemon/mpc) nếu cần CLI riêng.

## Troubleshooting (Windows)

### mpd.exe báo thiếu `libjack64.dll` hoặc DLL khác khi khởi động

Bản mpd.exe chính thức đôi khi link tới JACK audio dù không dùng tới. Cài [JACK Audio Connection Kit](https://jackaudio.org/downloads/) (chỉ cần cài, không cần chạy) để có DLL, hoặc dùng `audio_output type "winmm"` thay vì output khác cần JACK.

### `wasapi`/`winmm` báo không tìm thấy device

Tên device trên Windows có thể bị cắt ngắn/không khớp chính xác. Mở **Volume Mixer** (chuột phải biểu tượng loa ở taskbar → Open Volume Mixer) hoặc **Sound settings** để xem tên chính xác của thiết bị output, dán đúng vào `device "..."` trong `audio_output`. Nếu vẫn lỗi, bỏ hẳn dòng `device` để mpd tự dùng thiết bị mặc định của hệ thống.

### Chạy như Windows Service, log không ghi được / lỗi quyền

Service chạy dưới tài khoản hệ thống (LOCAL SERVICE) không có quyền ghi vào `C:\mpd\` mặc định. Vào **Properties** của thư mục `C:\mpd\` → **Security** → cấp quyền Full Control cho account chạy service (hoặc đơn giản hơn: dùng cách chạy qua Startup/Task Scheduler ở Bước 4 thay vì service thật, chạy dưới chính user account của bạn nên không dính lỗi quyền).
