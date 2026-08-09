# mpd + rmpc setup (macOS)

Cấu hình cá nhân cho [MPD](https://www.musicpd.org/) (Music Player Daemon) và [rmpc](https://github.com/mierak/rmpc) (terminal client, có album art) chạy trên macOS qua Homebrew.

## Cấu trúc repo

```
.
├── README.md
├── INSTALL.md          # hướng dẫn cài đặt từng bước
├── mpd.conf             # config của mpd (đặt tại ~/.config/mpd/mpd.conf)
└── rmpc/
    └── config.toml       # config của rmpc (đặt tại ~/.config/rmpc/config.toml)
```

Các file runtime của mpd (`database`, `log`, `state`, `pid`, `playlists/`) **không** được commit — chúng được tự sinh lại khi mpd chạy lần đầu.

## Thành phần

- **mpd** — daemon phát nhạc chạy nền, quản lý qua `brew services`, expose control qua TCP `127.0.0.1:6600`.
- **mpc** — CLI đơn giản để điều khiển mpd, dùng để test nhanh (`mpc play`, `mpc status`, ...).
- **rmpc** — TUI client kết nối tới mpd, giao diện chính để duyệt/phát nhạc hàng ngày.

## Điểm cần lưu ý trên macOS

Setup này có vài chỗ "gài" đặc thù macOS đã được xử lý sẵn trong `mpd.conf`, xem chi tiết + lý do trong [INSTALL.md](./INSTALL.md#troubleshooting--các-lỗi-đã-gặp):

1. `audio_output` phải dùng `type "osx"` (CoreAudio) — không phải `pulse` (PulseAudio không có sẵn trên macOS).
2. Tất cả đường dẫn trong `mpd.conf` dùng **đường dẫn tuyệt đối** (`/Users/xxx/...`) thay vì `~`, vì mpd chạy qua `launchd` (brew services) không có biến môi trường `HOME`.
3. `music_directory` **không được trỏ vào thư viện Apple Music** (`~/Music/Music/Media.localized`) — macOS chặn quyền đọc (TCC) khiến mpd không mở được file. Nhạc nên để ở một thư mục thường, ví dụ `~/Workspace/Music`.

## Cài đặt

Xem hướng dẫn chi tiết từng bước tại **[INSTALL.md](./INSTALL.md)**.

## License

Config cá nhân, tự do tham khảo/sao chép.
