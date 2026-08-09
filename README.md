# mpd + rmpc setup

Cấu hình cá nhân cho [MPD](https://www.musicpd.org/) (Music Player Daemon) và [rmpc](https://github.com/mierak/rmpc) (terminal client, có album art).

- **macOS** — cài qua Homebrew, đã setup và test trực tiếp.
- **Windows** — hướng dẫn qua WSL2 hoặc native Windows (best-effort, rmpc [chỉ chính thức hỗ trợ Linux](https://rmpc.mierak.dev)).

## Cấu trúc repo

```
.
├── README.md
├── INSTALL.md              # hướng dẫn cài đặt từng bước
├── mpd.conf                 # config của mpd (đặt tại ~/.config/mpd/mpd.conf)
└── rmpc/
    ├── config.ron           # config của rmpc (đặt tại ~/.config/rmpc/config.ron)
    └── themes/
        └── nord.ron          # theme Nord (đặt tại ~/.config/rmpc/themes/nord.ron)
```

Các file runtime của mpd (`database`, `log`, `state`, `pid`, `playlists/`) **không** được commit — chúng được tự sinh lại khi mpd chạy lần đầu.

> **Lưu ý về định dạng config của rmpc:** rmpc dùng định dạng **RON** (`config.ron`), không phải TOML/YAML. Nếu tự viết config tay, kiểm tra lại bằng `rmpc debuginfo` — dòng `Config path` phải trỏ đúng file, nếu là `None` nghĩa là rmpc không đọc được config và đang chạy 100% giá trị mặc định. File `rmpc/config.ron` trong repo này được sinh trực tiếp từ `rmpc config` (in ra toàn bộ default config) nên chắc chắn đúng schema.

Theme đang dùng: **Nord** (`rmpc/themes/nord.ron`, tham chiếu qua `theme: "nord"` trong `config.ron`). Yêu cầu terminal cài **Nerd Font** để hiện đúng icon — xem [INSTALL.md](./INSTALL.md#theme-nord) để biết cách cài và các lỗi thường gặp khi tự lấy theme từ docs (bản "latest/next" trên trang docs rmpc có thể không khớp schema với bản `rmpc` đang cài).

## Thành phần

- **mpd** — daemon phát nhạc chạy nền, quản lý qua `brew services`, expose control qua TCP `127.0.0.1:6600`.
- **mpc** — CLI đơn giản để điều khiển mpd, dùng để test nhanh (`mpc play`, `mpc status`, ...).
- **rmpc** — TUI client kết nối tới mpd, giao diện chính để duyệt/phát nhạc hàng ngày.

## Điểm cần lưu ý trên macOS

Setup này có vài chỗ "gài" đặc thù macOS đã được xử lý sẵn trong `mpd.conf`, xem chi tiết + lý do trong [INSTALL.md](./INSTALL.md#macos):

1. `audio_output` phải dùng `type "osx"` (CoreAudio) — không phải `pulse` (PulseAudio không có sẵn trên macOS).
2. Tất cả đường dẫn trong `mpd.conf` dùng **đường dẫn tuyệt đối** (`/Users/xxx/...`) thay vì `~`, vì mpd chạy qua `launchd` (brew services) không có biến môi trường `HOME`.
3. `music_directory` **không được trỏ vào thư viện Apple Music** (`~/Music/Music/Media.localized`) — macOS chặn quyền đọc (TCC) khiến mpd không mở được file. Nhạc nên để ở một thư mục thường, ví dụ `~/Workspace/Music`.

## Cài đặt

Xem hướng dẫn chi tiết từng bước tại **[INSTALL.md](./INSTALL.md)** — có cả macOS và Windows.

## License

[MIT](./LICENSE)
