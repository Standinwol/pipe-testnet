Dưới đây là hướng dẫn chi tiết, từng bước để bạn triển khai và khởi chạy POP Cache Node trên một VPS (Ubuntu 20.04+). Mỗi bước đều kèm lệnh cụ thể và giải thích:

---

## A. Chuẩn bị trước khi bắt đầu

1. **Yêu cầu**

   * Một VPS hoặc EC2 instance chạy Ubuntu 20.04+
   * Bạn có quyền SSH (private key `.pem`) và user `ubuntu` (hoặc user tương đương có sudo).
   * Đảm bảo đã mở port **80** và **443** trên firewall (Security Group hoặc UFW).

2. **SSH vào máy chủ**

   ```bash
   ssh -i /path/to/key.pem ubuntu@YOUR_EC2_PUBLIC_IP
   ```

   > Thay `YOUR_EC2_PUBLIC_IP` bằng IP public thực tế của VPS.

---

## B. Cập nhật hệ thống và cài đặt phụ thuộc

1. **Cập nhật & nâng cấp package**

   ```bash
   sudo apt update && sudo apt upgrade -y
   ```
2. **Cài các gói cần thiết**

   ```bash
   sudo apt install -y \
     libssl-dev \
     ca-certificates \
     tar \
     wget \
     libcap2-bin \
     jq
   ```

   * `libssl-dev`, `ca-certificates`: hỗ trợ SSL/TLS
   * `tar`, `wget`: giải nén, tải file
   * `libcap2-bin`: để setcap (cấp quyền bind port <1024)
   * `jq`: validate JSON

---

## C. Tối ưu mạng & tăng giới hạn file

1. **Tối ưu tham số TCP (network tuning)**
   Tạo file `/etc/sysctl.d/99-popcache.conf` với các giá trị tối ưu:

   ```bash
   sudo tee /etc/sysctl.d/99-popcache.conf <<EOF
   net.ipv4.ip_local_port_range = 1024 65535
   net.core.somaxconn = 65535
   net.ipv4.tcp_low_latency = 1
   net.ipv4.tcp_fastopen = 3
   net.ipv4.tcp_slow_start_after_idle = 0
   net.ipv4.tcp_window_scaling = 1
   net.ipv4.tcp_wmem = 4096 65536 16777216
   net.ipv4.tcp_rmem = 4096 87380 16777216
   net.core.wmem_max = 16777216
   net.core.rmem_max = 16777216
   EOF
   ```

   → Áp dụng cấu hình ngay:

   ```bash
   sudo sysctl --system
   ```

2. **Tăng giới hạn file descriptors**
   Tạo file `/etc/security/limits.d/popcache.conf`:

   ```bash
   sudo tee /etc/security/limits.d/popcache.conf <<EOF
   *    hard nofile 65535
   *    soft nofile 65535
   EOF
   ```

   → **Quan trọng:** Đăng xuất (`exit`) rồi SSH lại để cấu hình `limits` có hiệu lực.

---

## D. Tạo thư mục và cài đặt binary POP

1. **Tạo thư mục lưu trữ**

   ```bash
   sudo mkdir -p /opt/popcache/{logs,cache}
   sudo chown -R ubuntu:ubuntu /opt/popcache
   cd /opt/popcache
   ```
2. **Tải binary**

   ```bash
   wget https://download.pipe.network/static/pop-v0.3.0-linux-x64.tar.gz
   ```
3. **Giải nén & set quyền**

   ```bash
   tar -xzf pop-v0.3.0-linux-x64.tar.gz
   chmod 755 pop
   ```

---

## E. Tạo & cấu hình `config.json`

1. Mở file cấu hình:

   ```bash
   nano /opt/popcache/config.json
   ```

2. Dán nội dung mẫu, sửa lại theo thông tin của bạn:

   ```json
   {
     "pop_name":        "Ten_POP_Cua_Ban",
     "pop_location":    "Thanh-hoa, Viet-Nam",
     "invite_code":     "YOUR_INVITE_CODE",
     "server": {
       "host":          "0.0.0.0",
       "port":          443,
       "http_port":     80,
       "workers":       0
     },
     "cache_config": {
       "memory_cache_size_mb": 4096,
       "disk_cache_path":      "./cache",
       "disk_cache_size_gb":   100,
       "default_ttl_seconds":  86400,
       "respect_origin_headers": true,
       "max_cacheable_size_mb": 1024
     },
     "api_endpoints": {
       "base_url": "https://dataplane.pipenetwork.com"
     },
     "identity_config": {
       "node_name":      "Node-Cua-Toi",
       "name":           "Ten-Cua-Toi",
       "email":          "email.cua.toi@example.com",
       "website":        "https://website-cua-toi.com",
       "discord":        "discordUser#1234",
       "telegram":       "telegramUser",
       "solana_pubkey":  "YourSolanaPubkeyHere"
     }
   }
   ```

3. Lưu (Ctrl+X → Y → Enter).

4. **Validate JSON**:

   ```bash
   jq . /opt/popcache/config.json >/dev/null || echo "❌ Lỗi JSON!"
   ```

---

## F. Cấp quyền bind port 443 cho binary

Theo mặc định, non-root không bind port <1024 được. Dùng `setcap`:

```bash
sudo setcap 'cap_net_bind_service=+ep' /opt/popcache/pop
getcap /opt/popcache/pop
# Kết quả phải là: /opt/popcache/pop = cap_net_bind_service=ep
```

---

## G. Tạo & kích hoạt systemd service

1. **Tạo file service** `/etc/systemd/system/popcache.service`:

   ```bash
   sudo tee /etc/systemd/system/popcache.service <<EOF
   [Unit]
   Description=POP Cache Node
   After=network.target

   [Service]
   Type=simple
   User=ubuntu
   Group=ubuntu
   WorkingDirectory=/opt/popcache
   ExecStart=/opt/popcache/pop
   Restart=always
   RestartSec=5
   LimitNOFILE=65535
   StandardOutput=append:/opt/popcache/logs/stdout.log
   StandardError=append:/opt/popcache/logs/stderr.log
   Environment=POP_CONFIG_PATH=/opt/popcache/config.json

   [Install]
   WantedBy=multi-user.target
   EOF
   ```

2. **Nạp lại systemd, bật và khởi động service**:

   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable popcache
   sudo systemctl start popcache
   ```

---

## H. Kiểm tra trạng thái & xác thực

1. **Kiểm tra service**

   ```bash
   sudo systemctl status popcache
   ```

   → Trạng thái phải là **active (running)**.

2. **Kiểm tra cổng 443 lắng nghe**

   ```bash
   sudo ss -ltnp | grep ':443'
   ```

3. **Thử truy cập endpoint**

   ```bash
   curl -I http://YOUR_VPS_IP/
   curl -I https://YOUR_VPS_IP/ --insecure
   ```

   → HTTP 200 OK hoặc chuyển hướng.

4. **Xem log realtime**

   ```bash
   tail -f /opt/popcache/logs/stdout.log
   tail -f /opt/popcache/logs/stderr.log
   ```

---

## I. Bảo mật & vận hành lâu dài

1. **Firewall (UFW)**

   ```bash
   sudo ufw allow 80/tcp
   sudo ufw allow 443/tcp
   sudo ufw enable
   ```

2. **Cập nhật phiên bản mới**

   ```bash
   sudo systemctl stop popcache
   # upload binary pop mới vào /opt/popcache/pop
   sudo chmod 755 /opt/popcache/pop
   sudo systemctl start popcache
   ```

3. **Khởi động lại VPS**

   * Reboot không mất dữ liệu (EBS), systemd sẽ tự động khởi động lại service.

---

Chúc bạn triển khai thành công! 
