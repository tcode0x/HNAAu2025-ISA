# CSD TECH - File Server Setup Guide (VMware + Windows Server 2019)

## 1. Giới thiệu

Tài liệu này hướng dẫn từng bước để cài đặt và cấu hình File Server trên Windows Server 2019 khi chạy trong môi trường ảo hóa VMware (Workstation hoặc ESXi). Bao gồm thiết lập mạng nội bộ (VMnet2), đặt IP tĩnh, cài đặt role File Server, tạo thư mục chia sẻ, gán quyền NTFS, mở firewall và kiểm tra kết nối nội bộ.

---

## 2. Cấu hình mạng nội bộ (VMnet2)

### 2.1 Mục tiêu
Tạo mạng host-only riêng (VMnet2) giữa máy chủ và các máy ảo trong cùng môi trường để chia sẻ file mà không phụ thuộc Internet.

### 2.2 Các bước
1. Mở **VMware Workstation** → chọn **Edit > Virtual Network Editor**.
2. Chọn **Add Network...** → chọn **VMnet2**.
3. Thiết lập:
   - Connection type: **Host-only (connect VMs internally)**
   - DHCP: **Tắt (Disable)**
   - Subnet IP: `192.168.56.0`
   - Subnet Mask: `255.255.255.0`
4. Bấm **Apply** và **OK** để lưu cấu hình.

---

## 3. Thiết lập IP tĩnh cho Windows Server

1. Trong Windows Server, mở **Control Panel > Network and Sharing Center > Change adapter settings**.
2. Chuột phải vào **Ethernet (VMnet2)** → chọn **Properties**.
3. Chọn **Internet Protocol Version 4 (TCP/IPv4)** → **Properties**.
4. Cấu hình IP tĩnh:
   ```
   IP Address: 192.168.56.10
   Subnet Mask: 255.255.255.0
   Default Gateway: (để trống)
   Preferred DNS: 8.8.8.8
   ```
5. Lưu lại, rồi kiểm tra lại bằng:
   ```powershell
   ipconfig
   ```
   Kết quả mong muốn:
   ```
   IPv4 Address: 192.168.56.10
   ```

---

## 4. Cài đặt File Server Role

Mở PowerShell (Run as Administrator) và chạy:

```powershell
Install-WindowsFeature FS-FileServer -IncludeManagementTools
```

Kiểm tra lại bằng:

```powershell
Get-WindowsFeature FS-FileServer
```

Nếu hiển thị `Installed: True` là đã cài thành công.

---

## 5. Tạo thư mục chia sẻ

```powershell
New-Item -Path "D:\Share\TaiLieuChung" -ItemType Directory -Force
```

---

## 6. Gán quyền NTFS và tạo SMB Share

```powershell
$path = "D:\Share\TaiLieuChung"

# Gán quyền NTFS
icacls $path /grant "Everyone:(RX)" "Users:(M)" "Administrators:(F)" /T

# Tạo SMB Share
New-SmbShare -Name "TaiLieuChung" -Path $path -FullAccess "Administrators" -ChangeAccess "Users" -ReadAccess "Everyone"

# Bật Access-Based Enumeration
Set-SmbShare -Name "TaiLieuChung" -FolderEnumerationMode AccessBased
```

---

## 7. Mở Firewall và cấu hình Network Profile

```powershell
# Mở firewall cho SMB
Get-NetFirewallRule -DisplayGroup "File and Printer Sharing" | Set-NetFirewallRule -Enabled True

# Đặt network profile thành Private
Set-NetConnectionProfile -NetworkCategory Private
```

---

## 8. Kiểm tra dịch vụ SMB

```powershell
Get-Service LanmanServer
```

Nếu `Status` không phải `Running`, chạy:

```powershell
Start-Service LanmanServer
```

---

## 9. Kết nối từ máy client trong cùng VMnet2

Giả sử máy server có IP `192.168.56.10`.

### 9.1 Kiểm tra kết nối
Trên máy client (Windows hoặc Linux trong cùng VMnet2):
```
ping 192.168.56.10
```

### 9.2 Truy cập thư mục chia sẻ
Mở **Run (Win + R)** → nhập:
```
\\192.168.56.10\TaiLieuChung
```
Nếu được yêu cầu, đăng nhập bằng:
```
Username: Administrator
Password: (mật khẩu server)
```

---

## 10. Script tự động cấu hình File Server

Tạo file `Setup_FileServer.ps1` với nội dung:

```powershell
# CSD TECH - File Server Auto Setup Script

Install-WindowsFeature FS-FileServer -IncludeManagementTools

$sharePath = "D:\Share\TaiLieuChung"
if (!(Test-Path $sharePath)) { New-Item -Path $sharePath -ItemType Directory -Force }

icacls $sharePath /grant "Everyone:(RX)" "Users:(M)" "Administrators:(F)" /T

New-SmbShare -Name "TaiLieuChung" -Path $sharePath -FullAccess "Administrators" -ChangeAccess "Users" -ReadAccess "Everyone"

Get-NetFirewallRule -DisplayGroup "File and Printer Sharing" | Set-NetFirewallRule -Enabled True
Set-NetConnectionProfile -NetworkCategory Private

Set-SmbShare -Name "TaiLieuChung" -FolderEnumerationMode AccessBased

Write-Host "File Server ready at: \\$env:COMPUTERNAME\TaiLieuChung"
```

Chạy script:
```powershell
powershell.exe -ExecutionPolicy Bypass -File .\Setup_FileServer.ps1
```

---

## 11. Kiểm tra hoạt động

```powershell
Get-SmbShare
Test-NetConnection 192.168.56.10 -Port 445
```

Hoặc truy cập từ máy client:
```
\\192.168.56.10\TaiLieuChung
```

---

## 12. Troubleshooting

| Vấn đề | Nguyên nhân | Cách khắc phục |
|--------|--------------|----------------|
| Không ping được | Cấu hình VMnet hoặc firewall sai | Kiểm tra lại IP và VMnet2 |
| Network name cannot be found | IP sai hoặc dịch vụ SMB chưa chạy | Kiểm tra LanmanServer |
| Access Denied | Quyền NTFS chưa đúng | Sửa quyền NTFS bằng icacls |
| Không map được drive | Sai thông tin đăng nhập | Sử dụng lệnh `net use Z: \\192.168.56.10\TaiLieuChung /user:Administrator` |

---

## 13. Gợi ý mở rộng

- Tạo thêm thư mục chia sẻ cho từng team (IT, HR, Student, Public).
- Bật Shadow Copies để người dùng khôi phục file cũ.
- Cài File Server Resource Manager (FSRM) để giới hạn dung lượng hoặc chặn file rác.
- Sao lưu định kỳ ổ đĩa dữ liệu (D:) để tránh mất file.

---

## 14. Kết luận

Sau khi hoàn tất, File Server đã sẵn sàng sử dụng trong mạng nội bộ VMware. Các máy client trong cùng VMnet2 có thể truy cập bằng địa chỉ:
```
\\192.168.56.10\TaiLieuChung
```

Hệ thống này có thể mở rộng thành môi trường domain nội bộ, tích hợp với Active Directory, FSRM và DFS tùy theo nhu cầu đào tạo hoặc doanh nghiệp.
