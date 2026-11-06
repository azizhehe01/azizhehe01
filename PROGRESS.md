# Handover & Progress Document — Quickshell Chameleon Frosted Glass Widgets & Notification Server

> **Tujuan Dokumen Ini:** Menyimpan seluruh progres, arsitektur, lokasi file, dan cara kerja sistem *Desktop Widgets + Notification Popup* di Quickshell & Hyprland agar jika sesi/limit agent habis, agent berikutnya dapat langsung melanjutkan tanpa perlu riset ulang dari nol.

---

## 1. Ringkasan Proyek & Lokasi File Utama

Seluruh komponen widget dan sistem notifikasi disatukan di dalam satu folder/repository untuk memudahkan pembuatan *dotfiles*:
- **Direktori Utama**: `/home/azhehe/.config/quickshell/DesktopWidgetsApp/`
  - [`shell.qml`](file:///home/azhehe/.config/quickshell/DesktopWidgetsApp/shell.qml) — Entry point utama Quickshell yang memuat `DesktopWidgetsWindow` dan `NotificationPopupWindow`.
  - [`DesktopWidgetsWindow.qml`](file:///home/azhehe/.config/quickshell/DesktopWidgetsApp/DesktopWidgetsWindow.qml) — Bento Grid Desktop Widgets di sisi kiri layar (`WlrLayer.Bottom`, namespace `quickshell:desktop-widgets`).
  - [`NotificationPopupWindow.qml`](file:///home/azhehe/.config/quickshell/DesktopWidgetsApp/NotificationPopupWindow.qml) — Server & Popup Notifikasi native Quickshell di pojok kanan atas (`WlrLayer.Overlay`, namespace `quickshell:notifications`).
  - [`stats.py`](file:///home/azhehe/.config/quickshell/DesktopWidgetsApp/stats.py) — Service Python ringan (dijalankan tiap 2.5 detik oleh QML) untuk membaca lirik real-time (`LRCLIB` + `playerctl position`), cuaca (`wttr.in`), penggunaan disk (`/` dan `/home`), serta baterai (`/sys/class/power_supply/BAT*`).
  - [`README.md`](file:///home/azhehe/.config/quickshell/DesktopWidgetsApp/README.md) — Dokumentasi lengkap fitur, sumber data, instalasi, dan konfigurasi Hyprland.

- **Konfigurasi Hyprland (Lua Config di ML4W / Hyprland 0.56.2)**:
  - [`/home/azhehe/.config/hypr/conf/decorations/default.lua`](file:///home/azhehe/.config/hypr/conf/decorations/default.lua) — Konfigurasi GPU Dual-Kawase blur (`size = 10`, `passes = 3`, `xray = false`, `vibrancy = 0.25`) dan `hl.layer_rule` untuk `quickshell:desktop-widgets` serta `quickshell:notifications` dengan `ignore_alpha = 0.1`.
  - [`/home/azhehe/.config/hypr/conf/windowrules/default.lua`](file:///home/azhehe/.config/hypr/conf/windowrules/default.lua) — Definisi `hl.layer_rule` cadangan.
  - [`/home/azhehe/.config/ml4w/scripts/ml4w-autostart`](file:///home/azhehe/.config/ml4w/scripts/ml4w-autostart) — Script autostart yang mematikan `swaync` dan menjalankan `qs -p $HOME/.config/quickshell/DesktopWidgetsApp &`.

---

## 2. Apa Saja yang Sudah Selesai Dikerjakan (Completed)

### A. Real-Time GPU Chameleon Frosted Glass (100% Tanpa Tint Coklat & Real-Time)
- **Masalah Sebelumnya**:
  1. Widget sebelumnya memiliki tint coklat statis dan merender potongan gambar wallpaper (`Image`) di dalam kartu QML, sehingga ketika membuka aplikasi seperti Antigravity IDE, notifikasi malah menampilkan gambar wallpaper di atas window aplikasi (bukan mem-blur window aplikasi di belakangnya secara *real-time*).
  2. Nilai `ignore_alpha = 0.2` di Hyprland terlalu tinggi dibandingkan alpha `cardBg` (`0.16–0.20`), sehingga GPU blur Hyprland sempat terlewati.
- **Solusi yang Sudah Diterapkan**:
  - Menghapus layer `Image` wallpaper statis dari [`NotificationPopupWindow.qml`](file:///home/azhehe/.config/quickshell/DesktopWidgetsApp/NotificationPopupWindow.qml) dan [`DesktopWidgetsWindow.qml`](file:///home/azhehe/.config/quickshell/DesktopWidgetsApp/DesktopWidgetsWindow.qml).
  - Menggunakan permukaan kaca netral murni (`cardBg: Qt.rgba(0.0, 0.0, 0.0, 0.22)` / `0.24`) + *specular shine* (`LinearGradient`) + *1px glass border highlight* (`Qt.rgba(1, 1, 1, 0.14)`).
  - Menurunkan `ignore_alpha = 0.1` pada `hl.layer_rule` di [`/home/azhehe/.config/hypr/conf/decorations/default.lua`](file:///home/azhehe/.config/hypr/conf/decorations/default.lua) dengan `xray = false`, `size = 10`, `passes = 3`.
  - Hasilnya: Baik widget maupun notifikasi sekarang **100% di-blur secara real-time oleh GPU (Hyprland Dual-Kawase Blur)** mengikuti window atau wallpaper apa pun yang ada tepat di belakangnya.

### B. Sistem Notifikasi Native (`NotificationPopupWindow.qml`)
- Menggantikan `swaync` (service `swaync.service` sudah di-*mask* dan di-*stop* agar tidak merebut D-Bus `org.freedesktop.Notifications`).
- **Tanpa Tombol `✕` yang Mengganggu**: Tombol `✕` di pojok kanan atas sudah dihapus; klik di area kosong kartu langsung menutup (*dismiss*) notifikasi.
- **Isi Notifikasi Bisa Di-Scroll (`Flickable` + `WheelHandler`)**:
  - Tidak lagi dipotong 3 baris (`maximumLineCount: 3` sudah dihapus).
  - Pesan pendek tampil kompak sesuai tinggi teks; pesan panjang otomatis dibatasi maksimal `120px` dan bisa **di-scroll menggunakan mouse wheel / dua jari di touchpad** lengkap dengan indikator *slim frosted scrollbar* di sisi kanan.
  - Selama kursor berada di atas kartu notifikasi (saat membaca/scroll), timer *auto-dismiss* otomatis berhenti (*pause*).
- **Dukungan Tombol Aksi (D-Bus Actions) & Fokus Cepat Antigravity IDE**:
  - Menampilkan hingga 4 tombol aksi D-Bus (`Allow`, `Deny`, `Approve`, dll.) jika aplikasi pengirim menyertakan `notif.actions`, dan langsung memanggil `notif.actions[idx].invoke()` saat diklik.
  - Untuk notifikasi dari **Antigravity IDE**, otomatis menampilkan tombol dengan icon SVG monokrom putih ([`assets/rocket.svg`](file:///home/azhehe/.config/quickshell/DesktopWidgetsApp/assets/rocket.svg)) **`Buka Antigravity`**.
  - **Akurasi Multi-Window / Multi-Workspace (`focus_on_activate = true`)**: Ketika user membuka lebih dari 1 window Antigravity IDE di workspace berbeda (mis. Workspace 2 dan Workspace 4), klik tombol **`Buka Antigravity`** memanggil `nObj.actions[actIdx].invoke()` (D-Bus action `"default"`) **sebelum** menghapus objek notifikasi, tanpa ditimpa oleh selector generik `class:antigravity-ide`. Dipadukan dengan `misc = { focus_on_activate = true }` di [`/home/azhehe/.config/hypr/conf/decorations/default.lua`](file:///home/azhehe/.config/hypr/conf/decorations/default.lua), Electron mengirimkan sinyal Wayland `xdg_activation_v1` tepat dari `BrowserWindow` yang meminta izin sehingga Hyprland langsung pindah ke workspace dan window Antigravity IDE yang tepat.
  - Seluruh warna merah pada notifikasi *critical/permission* sudah dihapus agar 100% murni *neutral frosted glass* tanpa warna mencolok.

### D. Interactive Top Notch & Auto-Hide (`notch/NotchWindow.qml`)
- **Fitur Utama**:
  - Menggantikan statusbar bawaan ML4W dengan top notch interaktif di tengah atas layar.
  - Mata putih animasi dengan 2-layer parallax yang melacak kursor secara real-time via daemon IPC Hyprland ([`cursor.py`](file:///home/azhehe/.config/quickshell/DesktopWidgetsApp/notch/cursor.py)) dan kedipan mata otomatis.
  - OSD Volume gaya Nothing OS (icon dot-matrix speaker, 16-dot capsule progress bar, font display `Ndot57`).
  - **HDMI / Display Projection Manager ala Windows (`Super + P`)**:
    - Notch bermorfosis menjadi panel proyeksi Nothing OS dot-matrix (`displayBodyW: 460`, `displayNotchH: 84`).
    - 4 opsi dot-matrix: **PC ONLY**, **DUPLICATE**, **EXTEND**, dan **SECOND**.
    - Navigasi keyboard: tekan `Super + P` berulang kali untuk cycling opsi (seperti di Windows) atau klik langsung dengan mouse.
    - Dikelola oleh backend Python daemon [`display.py`](file:///home/azhehe/.config/quickshell/DesktopWidgetsApp/notch/display.py) dan IPC Hyprland `hyprctl keyword monitor ...`.
  - GPU-accelerated rendering (`QtQuick.Shapes`) tanpa jitter Wayland surface resize.
- **Auto-Hide & Full Window Space**:
  - `reserveSpace: false` (`exclusiveZone: 0`): jendela aplikasi mendapatkan 100% tinggi layar tanpa terpotong 38px.
  - `autoHideMode: "any_window"` (Default): notch otomatis meluncur naik dan bersembunyi kapan pun ada jendela terbuka/aktif (misal saat membuka terminal via `Super + Enter`, browser, dsb). Hanya tampil saat berada di desktop bersih tanpa window.
  - Saat mengubah volume (keyboard shortcut / wheel): notch otomatis turun menampilkan HUD volume Nothing OS secara instan, lalu bersembunyi kembali setelah timeout.
  - Hover-to-reveal: saat kursor diarahkan ke ujung paling atas layar (`cursorY <= 8`), notch tetap dapat diintip kapan saja.

- **Font Resmi Nothing (`assets/fonts/`)**:
  - [`Ndot55-Regular.otf`](file:///home/azhehe/.config/quickshell/DesktopWidgetsApp/assets/fonts/Ndot55-Regular.otf) & [`Ndot57-Regular.otf`](file:///home/azhehe/.config/quickshell/DesktopWidgetsApp/assets/fonts/Ndot57-Regular.otf) — Digunakan untuk angka display berukuran besar (Jam Digital & Suhu `28°`).
  - [`NType82-Regular.otf`](file:///home/azhehe/.config/quickshell/DesktopWidgetsApp/assets/fonts/NType82-Regular.otf) — Font sans-serif geometris resmi dari web `nothing.tech` untuk teks ukuran sedang/kecil (judul lagu, kondisi cuaca, lirik, judul & isi notifikasi).
  - [`LetteraMonoLL-Regular.otf`](file:///home/azhehe/.config/quickshell/DesktopWidgetsApp/assets/fonts/LetteraMonoLL-Regular.otf) — Font technical monospace resmi dari web `nothing.tech` untuk metadata berukuran kecil (tanggal, lokasi kota, `H/L` suhu, kelembapan & kecepatan angin, kapasitas GB disk, dan nama aplikasi di notifikasi).
- **Kartu Cuaca & Lokasi Gaya Nothing OS ([`DesktopWidgetsWindow.qml`](file:///home/azhehe/.config/quickshell/DesktopWidgetsApp/DesktopWidgetsWindow.qml#L254-L355) & [`stats.py`](file:///home/azhehe/.config/quickshell/DesktopWidgetsApp/stats.py#L73-L190))**:
  - Kiri atas: Icon cuaca dot-matrix putih ([`assets/weather-dot.svg`](file:///home/azhehe/.config/quickshell/DesktopWidgetsApp/assets/weather-dot.svg)).
  - Kanan atas: Suhu tertinggi & terendah harian (`H 29°` / `L 23°`).
  - Tengah: Angka suhu besar dengan font titik-titik `Ndot55` (`28°`).
  - Bawah: Kondisi cuaca (`DRIZZLE` / `PARTLY CLOUDY`), diikuti baris lokasi kota otomatis (`BANYUMAS` / sesuai geolokasi IP) di kiri dan kelembapan + angin (`74% · 7 KM/H`) di kanan.
- **Pencegahan Auto-Respawn `swaync`**:
  - Di [`/home/azhehe/.config/quickshell/StatusbarApp/SwayncModule.qml`](file:///home/azhehe/.config/quickshell/StatusbarApp/SwayncModule.qml), `swaync-client -swb` sudah diset `running: false` agar statusbar ML4W tidak memicu D-Bus me-respawn `swaync`.

### D. Interactive Angry Eyes Notch & Nothing OS Volume OSD (`notch/`)
- **Struktur Modular (`notch/`)**:
  - [`notch/NotchWindow.qml`](file:///home/azhehe/.config/quickshell/DesktopWidgetsApp/notch/NotchWindow.qml) — Window notch di tengah atas layar (`WlrLayer.Top`), bentuk body hitam rounded dengan lengkungan bahu atas halus (`Canvas`).
  - [`notch/cursor.py`](file:///home/azhehe/.config/quickshell/DesktopWidgetsApp/notch/cursor.py) — Daemon ringan Python yang membaca koordinat kursor global via Hyprland IPC socket (`cursorpos`) ~30fps ke stdout Quickshell.
- **Sistem Animasi 2-Layer Parallax & Tatapan Mata**:
  - **Bagian Putih (Sclera)**: Ikut bergeser dinamis ke arah kursor (±4.5px horizontal, ±2.8px vertikal) lengkap dengan rotasi sudut perspektif 2.5D sehingga bola mata terasa hidup dan tidak kaku.
  - **Pupil Hitam**: Bergerak lebih dalam di dalam bagian putih (efek kedalaman 3D).
  - **Alis Galak**: Menurunkan kelopak atas saat kursor berada di bawah layar untuk mempertajam ekspresi galak/fokus.
  - **Animasi Kedip**: Berkedip otomatis tiap 3–6 detik secara acak (termasuk sesekali *double-blink*).
- **Nothing OS Dot-Matrix Volume OSD (Dynamic Island Morphing)**:
  - Terintegrasi langsung dengan `Quickshell.Services.Pipewire` (`Pipewire.defaultAudioSink` + `PwObjectTracker`).
  - Ketika volume dinaikkan/diturunkan (lewat tombol keyboard multimedia `XF86Audio*`, scroll mouse di notch, atau pavucontrol):
    - Notch otomatis melebar secara mulus (*dynamic island expansion*) dari `168px` ke `248px` (`animatedBodyW`).
    - Mata berangsur fade-out dan digantikan oleh **Nothing OS Dot-Matrix Volume HUD**:
      1. Icon speaker dot-matrix putih ([`assets/volume-dot.svg`](file:///home/azhehe/.config/quickshell/DesktopWidgetsApp/assets/volume-dot.svg) / [`assets/volume-mute-dot.svg`](file:///home/azhehe/.config/quickshell/DesktopWidgetsApp/assets/volume-mute-dot.svg)).
      2. Track volume 16 dot-capsule segmented bar ala glyph Nothing Phone (dot menyala putih cerah sesuai level volume, dot sisa redup transparan).
      3. Angka persentase dot-matrix resmi Nothing OS (`Ndot57`, misal `65%` atau `MUTED`).
    - Setelah 1.8 detik hening, notch otomatis mengecil kembali dan sepasang mata galak aktif kembali.
  - Mendukung interaksi scroll langsung di atas notch untuk menaikkan/menurunkan volume serta klik kanan untuk toggle mute.

---

## 3. Catatan Penting untuk Agent Berikutnya (Tips & Gotchas)

1. **Cara Hot-Reload / Menjalankan Command di Host Tanpa `BypassSandbox`**:
   - Quickshell otomatis melakukan *hot-reload* setiap kali file `.qml` di `/home/azhehe/.config/quickshell/DesktopWidgetsApp/` disimpan.
   - Hyprland otomatis melakukan *hot-reload* setiap kali `/home/azhehe/.config/hypr/conf/decorations/default.lua` disimpan.
   - Jika perlu menjalankan perintah di host (seperti me-restart `qs` atau mengirim `notify-send`), tambahkan sementara `hl.exec_cmd("...")` di baris paling bawah `/home/azhehe/.config/hypr/conf/decorations/default.lua`, simpan, lalu hapus kembali baris `hl.exec_cmd` tersebut.
   - **JANGAN PERNAH** menggabungkan `pkill -f DesktopWidgetsApp` dan `qs -p .../DesktopWidgetsApp` dalam satu `sh -c` karena `pkill -f` akan membunuh subshell itu sendiri.
   - **JANGAN** memanggil `hyprctl` di dalam `os.execute()` pada file `.lua` Hyprland karena `os.execute()` memblokir event loop IPC Hyprland; selalu gunakan `hl.exec_cmd()`.
2. **Sintaks `Hyprland.dispatch` di Hyprland 0.56.2 Lua Mode**:
   - Di Hyprland 0.56.2 dengan konfigurasi Lua, `Hyprland.dispatch(...)` dari Quickshell mengevaluasi `return hl.dispatch(<arg>)` yang membutuhkan objek dispatcher `hl.dsp.*`:
     ```qml
     Hyprland.dispatch('hl.dsp.focus({ window = "class:antigravity-ide" })');
     ```
3. **Log Quickshell**:
   - Log runtime Quickshell saat ini diarahkan ke `/tmp/qs_notif.log` (bisa dibaca langsung dengan `read_file` untuk mengecek apakah ada error QML).

