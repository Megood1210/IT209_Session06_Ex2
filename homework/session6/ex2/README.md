## 1. Tạo nhóm và người dùng
```bash
groupadd devops-admin
useradd -m -s /bin/bash deployer
usermod -aG devops-admin deployer
```
- `groupadd devops-admin`: tạo nhóm mới.
- `useradd -m -s /bin/bash deployer`: tạo user `deployer` kèm thư mục home + shell bash.
- `usermod -aG devops-admin deployer`: thêm `deployer` vào nhóm `devops-admin` (`-a` = append, không xoá nhóm cũ).

Kiểm tra user đã vào nhóm:
```bash
groups deployer
```

## 2. Cấu hình sudoers (dùng drop-in file an toàn)
Thay vì sửa trực tiếp `/etc/sudoers` bằng `visudo`, tạo file riêng trong `/etc/sudoers.d/`
(cách này được sudo khuyến nghị, an toàn, không phá file gốc):

```bash
echo '%devops-admin ALL=(ALL) NOPASSWD: /usr/bin/systemctl start *, /usr/bin/systemctl stop *, /usr/bin/systemctl restart *, /usr/bin/systemctl status *' > /etc/sudoers.d/devops-admin
chmod 440 /etc/sudoers.d/devops-admin
visudo -c
```
- Dấu `%` biểu thị đây là **Group** (không phải user).
- `NOPASSWD:` = không hỏi mật khẩu khi chạy các lệnh được liệt kê.
- `chmod 440`: quyền chuẩn bắt buộc cho file sudoers.
- `visudo -c`: kiểm tra cú pháp toàn bộ sudoers -> phải báo "parsed OK".

Dòng cấu hình đã thêm:
```
%devops-admin ALL=(ALL) NOPASSWD: /usr/bin/systemctl start *, /usr/bin/systemctl stop *, /usr/bin/systemctl restart *, /usr/bin/systemctl status *
```

## 3. Kiểm tra
```bash
su - deployer
sudo -l
sudo systemctl restart cron
exit
```

## 4. Kết quả thực tế

### sudo -l (khi đang là deployer)
```
Matching Defaults entries for deployer on fisher-9126:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin,
    use_pty

User deployer may run the following commands on fisher-9126:
    (ALL) NOPASSWD: /usr/bin/systemctl start *, /usr/bin/systemctl stop *,
        /usr/bin/systemctl restart *, /usr/bin/systemctl status *
```

### sudo systemctl restart cron
```
deployer@fisher-9126:~$ sudo systemctl restart cron
(chay xong, KHONG hoi mat khau, khong bao loi)
```

### Xác nhận cron chạy lại (sudo systemctl status cron)
```
 cron.service - Regular background program processing daemon
     Loaded: loaded (/lib/systemd/system/cron.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2026-10-06 08:15:18 +07
   Main PID: 72060 (cron)
```

## 5. Giải thích bảo mật
- Chỉ cấp đúng 4 nhóm lệnh `systemctl` thay vì toàn quyền root -> nguyên tắc **least privilege**.
- Phân quyền theo **nhóm** (`%devops-admin`) thay vì từng user -> dễ quản lý, thêm người chỉ cần đưa vào nhóm.
- `NOPASSWD` giúp tự động hoá (script deploy) chạy được mà không kẹt ở bước nhập mật khẩu,
  nhưng chỉ giới hạn trong các lệnh an toàn đã liệt kê.
