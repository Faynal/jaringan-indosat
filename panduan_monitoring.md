# Panduan Monitoring Jaringan 4G (Termux)

Panduan ini dibuat untuk memantau kualitas jaringan Indosat 4G secara otomatis, mencatat Latensi (Ping), Jitter, Packet Loss, serta Kecepatan Download/Upload.

## 1. Persiapan & Instalasi Awal
Jika Termux baru di-install ulang, jalankan perintah berikut:

```bash
# Update repositori
pkg update && pkg upgrade -y

# Install paket yang dibutuhkan
pkg install tmux curl bc -y

```
## 2. Membuat Skrip Utama (netmon.sh)

Skrip ini akan melakukan ping, tes download, dan tes upload setiap 5 menit sekali.

Buat file dengan perintah: 
```
nano ~/netmon.sh
```
Lalu tempel kode berikut:
```
#!/data/data/com.termux/files/usr/bin/bash
# netmon.sh - Monitor LENGKAP (Tanpa Termux:API)
# Output: ~/netmon.log

TARGET="${1:-8.8.8.8}"
COUNT="${2:-5}"
INTERVAL="${3:-300}"
LOG="$HOME/netmon.log"
DOWN_URL="https://speed.cloudflare.com/__down?bytes=5000000"
UP_URL="https://speed.cloudflare.com/__up"

touch "$LOG"

while true; do
    TIME=$(date '+%H:%M')

    # 1. Ping: RTT + loss
    PING_OUT=$(ping -c "$COUNT" -q "$TARGET" 2>/dev/null)
    RTT_LINE=$(echo "$PING_OUT" | grep -E "rtt|round-trip")
    LOSS_LINE=$(echo "$PING_OUT" | grep -oE '[0-9]+(\.[0-9]+)?% packet loss')

    if [ -n "$RTT_LINE" ]; then
        RTT=$(echo "$RTT_LINE" | cut -d= -f2 | awk '{print $1}')
        LOSS=$(echo "$LOSS_LINE" | grep -oE '[0-9]+(\.[0-9]+)?')
    else
        RTT="0/0/0/0"
        LOSS="100"
    fi

## 2. Download speed
    DOWN=$(curl -o /dev/null -s -w '%{speed_download}' --max-time 30 "$DOWN_URL" 2>/dev/null)
    [ -z "$DOWN" ] && DOWN="0"

    # 3. Upload speed
    UP=$(dd if=/dev/zero bs=1024 count=1024 2>/dev/null | \
         curl -o /dev/null -s -w '%{speed_upload}' --max-time 30 \
         -X POST --data-binary @- "$UP_URL" 2>/dev/null)
    [ -z "$UP" ] && UP="0"

    # 4. Tulis ke log
    printf "%s  %s ms  %s%%  %s  %s\n" \
        "$TIME" "$RTT" "$LOSS" "$DOWN" "$UP" >> "$LOG"

    sleep "$INTERVAL"
done

```
Simpan dengan Ctrl+O, Enter, lalu Ctrl+X.
Beri izin eksekusi: chmod +x ~/netmon.sh

## 3. Membuat Launcher (start_netmon.sh)

Skrip ini berfungsi menyalakan wake-lock dan menjalankan netmon.sh di background (tmux).

Buat file: nano ~/start_netmon.sh
Tempel kode ini:
```
#!/data/data/com.termux/files/usr/bin/bash
termux-wake-lock

if tmux has-session -t netmon 2>/dev/null; then
    tmux attach -t netmon
else
    tmux new-session -d -s netmon "~/netmon.sh"
    echo "✅ Pemantauan berjalan di background."
    echo "   Cek log: cat ~/netmon.log"
    echo "   Hentikan: tmux kill-session -t netmon"
fi

```
Simpan, lalu beri izin: chmod +x ~/start_netmon.sh

## 4. Membuat Alat Filter (netbad)

Alat untuk memunculkan hanya data yang bermasalah (Max > 100ms, Jitter > 30ms, Loss > 0%).

```
cat << 'INNER' > ~/netbad
#!/data/data/com.termux/files/usr/bin/bash
awk '{split($2,a,"/"); if(a[3]+0>100 || a[4]+0>30 || $3+0>0) print}' ~/netmon.log
INNER

chmod +x ~/netbad
mv ~/netbad $PREFIX/bin/

```
## 5. Cara Penggunaan Sehari-hari

Mulai Memantau:
```
~/start_netmon.sh
```
Jika kosong = koneksi sehat. Jika ada baris keluar = ada lonjakan/lag.

Melihat Semua Data Log:
```
cat ~/netmon.log
```
Menghentikan Pemantauan:
```
tmux kill-session -t netmon
termux-wake-unlock
```

Catatan Penting:

1. Skrip memakan kuota sekitar ~72 MB per jam (karena tes download/upload tiap 5 menit).
2. Format Log: Jam | min/avg/max/jitter ms | Loss% | Down_Bps | Up_Bps


**3. Simpan File**
Setelah teks di atas di-paste ke dalam `nano`:
*   Tekan **`Ctrl + O`** (Simpan)
*   Tekan **`Enter`** (Konfirmasi nama file)
*   Tekan **`Ctrl + X`** (Keluar)

Sekarang file `panduan_monitoring.md` sudah rapi dan tersimpan dengan format yang benar! Anda bisa membacanya dengan perintah `cat ~/panduan_monitoring.md`.
