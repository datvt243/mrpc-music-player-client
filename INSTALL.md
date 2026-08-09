# Hướng dẫn cài đặt (macOS)

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
cp ~/.config/mpd/rmpc/config.toml ~/.config/rmpc/config.toml
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

rmpc sẽ tự kết nối tới `127.0.0.1:6600` (theo `~/.config/rmpc/config.toml`) và hiển thị thư viện vừa quét.

---

## Troubleshooting — các lỗi đã gặp

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
