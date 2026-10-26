# Pentest Lab GUI (Kali Linux)

Ứng dụng giao diện Python Tkinter để chạy kiểm thử bảo mật trong **phòng lab được phép**:
Nmap, SQLMap, Hydra và bộ DoS gồm **hping3 SYN flood**, ApacheBench, Slowloris.

> HTTP flood (GoldenEye) đã được loại bỏ. Hping3 là công cụ DoS chính và chạy
> lệnh cố định `hping3 -S --flood -p 80 <IP>`.

## Quy trình sử dụng

1. Nhập **IP mục tiêu** vào ô trên cùng → bấm **⚡ Ping & Khóa IP**.
2. App chạy `ping -c 4 -W 2 <IP>`:
   - Ping được → hiện **"● IP OK - CÓ KẾT NỐI"** và khóa IP (IPLOCK).
   - Không ping được → hiện **"● KHÔNG CÓ KẾT NỐI"**, không cho chạy lệnh tấn công.
3. Sau khi khóa IP, mọi tab tự dùng IP đó — không cần nhập lại.

## Các lệnh được sử dụng

### Nmap

- **Quét SYN toàn port, tốc độ cao:**
  ```bash
  sudo nmap -sS -T4 -p- --min-rate 1000 <IP>
  ```
  `-sS` SYN scan, `-T4` tăng tốc, `-p-` quét đủ 65535 port, `--min-rate 1000`
  gửi tối thiểu 1000 gói/giây.

- **Quét toàn diện:**
  ```bash
  sudo nmap -A <IP>
  ```
  `-A` đã bao gồm `-sV` (nhận diện phiên bản dịch vụ), OS detection, script scan
  mặc định và traceroute — không cần thêm `-sV`.

### SQLMap

```bash
sqlmap -u "http://<IP>/DVWA/vulnerabilities/sqli/?id=1&Submit=Submit" \
       --dbs --level 3 --batch
```

- URL mẫu dùng `{IP}`, app tự thay bằng IP đã khóa.
- Có thể nhập thêm cookie (ví dụ `PHPSESSID=...; security=low`) trong tab SQLMap.
- App kiểm tra trước URL: cảnh báo nếu trả về 404 hoặc bị chuyển về `login.php`.

### Hydra

```bash
hydra -l admin -P <wordlist> -s <port> -t 4 -f -I <IP> http-post-form \
      "/DVWA/login.php:username=^USER^&password=^PASS^&Login=Login:S=Logout"
```

- Chọn wordlist trên máy Kali, ví dụ `/usr/share/wordlists/rockyou.txt`
  (giải nén bằng `sudo gunzip /usr/share/wordlists/rockyou.txt.gz` nếu cần).
- Port `443` tự thêm `-S` (HTTPS).
- **DVWA + CSRF:** app tự GET trang login để lấy `user_token` + `PHPSESSID`, ép
  `security=low`, chạy Hydra với `G=0` và điều kiện `F=login.php`, rồi tự dừng khi
  thấy mật khẩu đúng — kể cả khi `-f` không hoạt động như mong đợi.

### DoS

- **Hping3 - SYN Flood:**
  ```bash
  sudo hping3 -S --flood -p 80 <IP>
  ```
  - `-S` gửi gói SYN, `--flood` gửi liên tục hết tốc lực, `-p 80` nhắm port 80.
  - App dùng cố định lệnh trên (không thêm `-c`, `-i` hay `--rand-source`).
  - `Tự dừng sau (giây)` mặc định `0` = chạy vô hạn; đặt số giây hoặc bấm
    **⏹ Dừng (Kill)** để kết thúc.

- **ApacheBench:**
  ```bash
  ab -n 1000 -c 10 http://<IP>/
  ```
  `-n` tổng số request, `-c` số request đồng thời, `-k` keep-alive (tùy chọn).

- **Slowloris:**
  ```bash
  slowloris <IP> -p 80 -s 500
  ```
  `-s` số socket giữ mở đồng thời.

## Cài đặt trên Kali

```bash
sudo apt update
sudo apt install -y nmap sqlmap hydra hping3 apache2-utils

# Slowloris (nếu chưa có)
sudo test -d /opt/slowloris || \
  sudo git clone https://github.com/gkbrk/slowloris.git /opt/slowloris
sudo chmod +x /opt/slowloris/slowloris.py
sudo ln -sf /opt/slowloris/slowloris.py /usr/local/bin/slowloris
```

Tab **🛠️ Quản lý Tools** hiển thị công cụ nào đã cài / còn thiếu và nút sao chép
lệnh cài đặt. Trạng thái được làm mới khi đổi tab hoặc bấm nút **↻**, nên không
cần khởi động lại app sau khi cài tool.

### Slowloris bị báo thiếu

Kiểm tra trong terminal:

```bash
command -v slowloris
# Kỳ vọng: /usr/local/bin/slowloris

slowloris --help
```

Nếu `command -v` không trả về gì, tạo lại liên kết và cấp quyền thực thi:

```bash
sudo chmod +x /opt/slowloris/slowloris.py
sudo ln -sf /opt/slowloris/slowloris.py /usr/local/bin/slowloris
```

Sau đó bấm **↻** ở khối "CÔNG CỤ HỆ THỐNG" hoặc chuyển tab để app quét lại.
App cũng tự tìm Slowloris tại `/opt/slowloris/slowloris.py` nếu lệnh chưa có
trong `PATH`.

## Chạy ứng dụng

```bash
sudo python3 pentest_lab_gui.py
```

## Ghi chú vận hành

- `nmap -sS`, `nmap -A` (OS detection) và hping3 cần quyền root. Nếu app không chạy
  bằng root, tick ô **"Tự động thêm sudo cho lệnh root"**; app sẽ hỏi mật khẩu sudo
  qua hộp thoại nếu cần (mật khẩu chỉ giữ trong RAM phiên chạy).
- Slowloris không bắt buộc root, nhưng cần kết nối được tới IP mục tiêu và port
  đang mở. Nếu chạy trong lab có giới hạn socket, tăng `ulimit -n` trước khi chạy app.
- Nút **⏹ Dừng** kill cả cây tiến trình (kể cả tiến trình con của `sudo`) bằng
  process group + SIGKILL dự phòng, nên mọi lệnh DoS dừng thật sự.
- Đóng cửa sổ app cũng tự kill tiến trình DoS đang chạy.
- Output được ép không đệm (`stdbuf`, `PYTHONUNBUFFERED=1`) nên log hiện theo thời gian thực.
- ApacheBench chạy trong thread riêng, output đọc qua `queue.Queue` nên giao diện
  không bị treo trong lúc test.
- Khi hping3 chạy `--flood`, target có thể không thấy phản hồi và hping3 không in
  từng gói — đây là bình thường; bấm **⏹ Dừng (Kill)** hoặc đặt `Tự dừng sau`.
- Chỉ dùng Hping3/ApacheBench/Slowloris trên **server lab cô lập**; các công cụ này
  có thể làm treo dịch vụ.

## Lưu ý an toàn & pháp lý

- Chỉ chạy trên hệ thống bạn sở hữu hoặc có **giấy phép thử nghiệm bằng văn bản**.
- Không chạy DoS trên mạng chung, mạng công ty hoặc mạng trường ngoài phạm vi lab.
- Kết quả Suricata (`event_type: flow`, `sql_keyword_count`, `login_fail_count`,
  `suricata_alert_count`...) nên được quan sát trên server IDS lab riêng.
