# FRPC Expiration Feature

## ภาพรวม

การแก้ไขฝั่ง frpc เพื่อรองรับ **expire timestamp** ที่ส่งมาจาก frps

## การเปลี่ยนแปลงที่สำคัญ

### 1. Message Handling
- แก้ไข `handleNewProxyResp()` เพื่อรับ expire timestamp จาก server
- แสดง log พร้อม expire timestamp เมื่อ proxy เริ่มต้นสำเร็จ

### 2. Proxy Status Management
- เพิ่ม `ExpireAt` field ใน `WorkingStatus` struct
- แก้ไข `SetRunningStatus()` เพื่อรับและเก็บ expire timestamp
- เพิ่ม `GetExpireAt()` และ `IsExpired()` methods

### 3. API Endpoints
- เพิ่ม `ExpireAt` field ใน `ProxyStatusResp` สำหรับ API response
- เพิ่ม `/api/expired` endpoint เพื่อดู proxy ที่หมดอายุ

## การใช้งาน

### 1. การแสดงผลใน Console Log

เมื่อ proxy เริ่มต้นสำเร็จ frpc จะแสดง log ดังนี้:

```
[INFO] [web01] start proxy success to [web01.example.com:80], expires at 1719999999
```

### 2. การดู Status ผ่าน API

#### GET /api/status
```bash
curl http://localhost:7400/api/status
```

Response จะมี `expire_at` field:
```json
{
  "http": [
    {
      "name": "web01",
      "type": "http",
      "status": "running",
      "remote_addr": "web01.example.com:80",
      "expire_at": 1719999999
    }
  ]
}
```

#### GET /api/expired
```bash
curl http://localhost:7400/api/expired
```

Response:
```json
{
  "expired_proxies": ["web01", "web02"],
  "count": 2
}
```

### 3. การตรวจสอบ Expiration ใน Code

```go
// ตรวจสอบ proxy ที่หมดอายุ
expiredProxies := proxyManager.GetExpiredProxies()
for _, name := range expiredProxies {
    fmt.Printf("Proxy %s has expired\n", name)
}

// ตรวจสอบ proxy เฉพาะตัว
status, exists := proxyManager.GetProxyStatus("web01")
if exists && status.ExpireAt > 0 {
    if time.Now().Unix() >= status.ExpireAt {
        fmt.Printf("Proxy %s has expired\n", status.Name)
    }
}
```

## การทำงาน

### Flow การทำงาน
1. **frps** ส่ง `NewProxyResp` พร้อม `expire_at` field
2. **frpc** รับ expire timestamp และเก็บไว้ใน proxy status
3. **frpc** แสดง log พร้อม expire timestamp
4. **API** สามารถแสดง expire timestamp ได้
5. **Client code** สามารถตรวจสอบ proxy ที่หมดอายุได้

### การตรวจสอบ Expiration
- frpc จะเก็บ expire timestamp ไว้ใน memory
- สามารถตรวจสอบผ่าน API หรือ code ได้
- ไม่มีการลบ proxy อัตโนมัติในฝั่ง client (server จะจัดการเอง)

## ข้อควรระวัง

1. **Backward Compatibility**: การเปลี่ยนแปลงนี้ backward compatible
2. **Memory Storage**: Expire timestamp เก็บใน memory เท่านั้น
3. **No Auto Cleanup**: Client ไม่ลบ proxy ที่หมดอายุอัตโนมัติ
4. **API Access**: ต้องมี admin API enabled (`adminAddr` ใน config)

## การทดสอบ

1. รัน frps พร้อม plugin ที่ส่ง expire timestamp
2. รัน frpc พร้อม proxy configuration
3. ตรวจสอบ log เพื่อดู expire timestamp
4. เรียก API `/api/status` เพื่อดู expire timestamp
5. เรียก API `/api/expired` เพื่อดู proxy ที่หมดอายุ

## ตัวอย่าง Configuration

### frpc.toml
```toml
serverAddr = "127.0.0.1"
serverPort = 7000
adminAddr = "127.0.0.1"
adminPort = 7400

[[proxies]]
name = "web01"
type = "http"
localPort = 8080
subdomain = "web01"  # จะถูกแทนที่ด้วย subdomain จาก plugin
```

### การเรียก API
```bash
# ดู status ทั้งหมด
curl http://127.0.0.1:7400/api/status

# ดู proxy ที่หมดอายุ
curl http://127.0.0.1:7400/api/expired
``` 