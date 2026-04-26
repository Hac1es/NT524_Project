# Hướng dẫn Thu thập Log AWS CloudTrail vào ELK Stack

## Tổng quan Kiến trúc

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              AWS Cloud                                       │
│  ┌──────────────┐     ┌──────────────┐     ┌────────────────────────────┐   │
│  │  IAM/SSO     │────>│  CloudTrail  │────>│  S3 Bucket                 │   │
│  │  Sign-in     │     │  (Trail)     │     │  aws-cloudtrail-logs-xxx   │   │
│  └──────────────┘     └──────────────┘     └─────────────┬──────────────┘   │
│                                                          │                   │
└──────────────────────────────────────────────────────────┼───────────────────┘
                                                           │ Pull logs
                                                           ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                           ELK Stack (Ubuntu)                                 │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐                 │
│  │  Logstash    │────>│ Elasticsearch│────>│   Kibana     │                 │
│  │  (S3 Input)  │     │              │     │              │                 │
│  └──────────────┘     └──────────────┘     └──────────────┘                 │
└──────────────────────────────────────────────────────────────────────────────┘
```

**Luồng dữ liệu:**

1. User đăng nhập AWS qua IAM Identity Center (SSO) hoặc Console
2. CloudTrail ghi lại tất cả API calls và management events
3. Logs được lưu vào S3 Bucket dạng JSON (nén `.gz`)
4. Logstash pull logs từ S3 mỗi 2 phút (interval: 120s)
5. Filter xử lý, chuẩn hóa và đưa vào Elasticsearch

---

## Phần 1: Cấu hình AWS CloudTrail

### Bước 1.1: Tạo Trail mới

1. Đăng nhập **AWS Console** → **CloudTrail** → **Trails** → **Create trail**

2. **Trail name**: `security-audit-trail`; **Multi-region trail**: `yes`

3. **Storage location**:

   - ✅ Create new S3 bucket (hoặc chọn bucket có sẵn)
   - Bucket name: `aws-cloudtrail-logs-<account-id>-<random>` (tự động tạo)

4. **Log file SSE-KMS encryption**: Bỏ chọn (để đơn giản)

5. **CloudWatch Logs**: Bỏ qua (không cần vì đã đẩy về ELK)

6. Click **Next**

### Bước 1.2: Chọn Events cần log

1. **Event type**:

✅ Management events, còn lại bỏ

2. **Management events**:

✅ API activity: Write-only, còn lại bỏ

3. Click **Next** → **Create trail**

### Bước 1.3: Xác nhận S3 Bucket và Path

Sau khi tạo, note lại thông tin:

- **Bucket name**: `aws-cloudtrail-logs-803176059654-21c368a3` (ví dụ)
- **Prefix path**: `AWSLogs/<org-id>/<account-id>/CloudTrail/<region>/`

Logs sẽ được lưu theo cấu trúc:

```
s3://aws-cloudtrail-logs-xxx/
└── AWSLogs/
    └── o-2ml19p4vgt/           # Organization ID (nếu có)
        └── 803176059654/       # Account ID
            └── CloudTrail/
                └── ap-southeast-1/   # Region
                    └── 2026/01/09/   # Date
                        └── xxx.json.gz
```

---

## Phần 2: Tạo IAM User cho Logstash

### Bước 2.1: Tạo IAM Policy

1. **IAM** → **Policies** → **Create policy**

2. Chọn tab **JSON** và paste:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::aws-cloudtrail-logs-803176059654-21c368a3"
    },
    {
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::aws-cloudtrail-logs-803176059654-21c368a3/*"
    }
  ]
}
```

3. **Policy name**: `CloudTrail-Logstash`

### Bước 2.2: Tạo IAM User

1. **IAM** → **Users** → **Create user**
2. **User name**: `logstashl`
3. **Attach policies**: Chọn `CloudTrail-Logstash`
4. **Create user**

### Bước 2.3: Tạo Access Key

1. Vào user vừa tạo → **Security credentials** → **Create access key**
2. Chọn **Application running outside AWS**
3. **Download .csv file**

---

## Phần 3: Cấu hình Logstash trên Ubuntu

### Bước 3.1: Tạo file AWS Credentials

```bash
# Tạo file chứa credentials
sudo nano /etc/logstash/aws.env
```

Nội dung:

```env
AWS_ACCESS_KEY_ID=<your-access-key>
AWS_SECRET_ACCESS_KEY=<your-secret-key>
AWS_DEFAULT_REGION=ap-southeast-1
```

Phân quyền:

```bash
sudo chown logstash:logstash /etc/logstash/aws.env
sudo chmod 600 /etc/logstash/aws.env
```

### Bước 3.3: Cấu hình Logstash Input

Mở file `/etc/logstash/conf.d/01-input.conf` và **bỏ comment** phần S3:

```conf
input {
  # Beats input cho Keycloak/Keystone
  beats {
    port => 5044
    ssl => true
    ssl_certificate => "/etc/logstash/certs/logstash.crt"
    ssl_key => "/etc/logstash/certs/logstash.key"
    ssl_client_authentication => "none"
  }

  # AWS CloudTrail từ S3
  s3 {
    bucket => "aws-cloudtrail-logs-803176059654-21c368a3"
    region => "ap-southeast-1"
    prefix => "AWSLogs/o-2ml19p4vgt/803176059654/CloudTrail/ap-southeast-1/"
    codec => "json"
    interval => 120          # Pull mỗi 2 phút
    delete => false          # Không xóa file sau khi đọc
    # sincedb_path => "/var/lib/logstash/sincedb_s3"  # Track đã đọc file nào
  }
}
```

### Bước 3.5: Filter xử lý AWS Logs

File `/etc/logstash/conf.d/30-filter-aws.conf` đã được cấu hình để:

1. **Nhận diện log AWS**: Kiểm tra có mảng `Records` (đặc trưng CloudTrail)
2. **Lọc event quan trọng**:
   - `UserAuthentication` từ `signin.amazonaws.com` (Console sign-in)
   - `Authenticate` từ `sso.amazonaws.com` (SSO Portal sign-in)
3. **Chuẩn hóa (Normalization)**: Map sang schema chung với các trường:
   - `user_id`, `ip`, `event_type`, `outcome`, `user_agent`
   - `idp` = Keycloak, `sp` = AWS_IAM_Identity_Center
4. **GeoIP Lookup**: Xác định vị trí địa lý từ IP
5. **Cleanup**: Xóa dữ liệu thô để tiết kiệm storage

---

## Phần 4: Khởi động và Kiểm tra

### Bước 4.1: Restart Logstash

```bash
sudo systemctl restart logstash
sudo systemctl status logstash
```

### Bước 4.2: Kiểm tra logs

```bash
# Xem log realtime
sudo tail -f /var/log/logstash/logstash-plain.log

# Tìm lỗi S3
sudo grep -i "s3\|error" /var/log/logstash/logstash-plain.log
```

### Bước 4.3: Test thủ công

1. Đăng nhập AWS Console hoặc SSO Portal
2. Đợi 2-5 phút để CloudTrail ghi log vào S3
3. Đợi thêm 2 phút để Logstash pull
4. Kiểm tra trên Kibana:

```
GET auth-logs-*/_search
{
  "query": {
    "match": {
      "sp": "AWS_IAM_Identity_Center"
    }
  }
}
```
