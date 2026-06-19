# Báo cáo Minh chứng Hoàn thành Bài thực hành (Lab W10)

Tài liệu này tổng hợp toàn bộ các kết quả và minh chứng hình ảnh chứng minh hệ thống GitOps, bảo mật CI/CD, cơ chế đồng bộ Secret và kiểm duyệt chính sách (Gatekeeper Policies) đã được triển khai hoàn chỉnh và hoạt động ổn định.

---

## 📸 PHẦN 1: HƯỚNG DẪN CHI TIẾT & GỢI Ý CHỤP ẢNH MINH CHỨNG

Để hoàn thành bài báo cáo thực hành một cách tốt nhất, dưới đây là gợi ý các ảnh chụp màn hình cần có và cách thực hiện:

### 1. Trạng thái các ứng dụng trên ArgoCD Dashboard
*   **Cách chụp:** Mở trình duyệt web truy cập vào ArgoCD UI, chụp toàn bộ màn hình Dashboard hiển thị trạng thái của các ứng dụng như `app-api`, `gatekeeper`, `eso-config`,... tất cả phải hiển thị màu xanh lá cây tượng trưng cho trạng thái **`Synced`** (Đã đồng bộ) và **`Healthy`** (Khỏe mạnh).
*   **Minh chứng:**
![alt text](<image/image copy 12.png>)
### 2. AWS Secrets Manager lưu trữ thông tin SMTP Gmail
*   **Cách chụp:** Đăng nhập vào AWS Console, tìm kiếm dịch vụ **Secrets Manager**, chụp chi tiết secret `email-password` dùng để lưu trữ an toàn mật khẩu ứng dụng Gmail mà không lo bị lộ trên Git.
*   **Minh chứng:**
![alt text](<image/image copy 13.png>)
### 3. Đồng bộ Secret bằng External Secrets Operator (ESO)
*   **Cách chụp:** Mở terminal chạy lệnh kiểm tra trạng thái đồng bộ:
    ```bash
    kubectl get secretstore,externalsecret -n monitoring
    ```
    Chụp kết quả hiển thị tình trạng `SecretStore` kết nối AWS thành công (`STATUS: Valid`) và `ExternalSecret` tạo ra K8s Secret thành công.
*   **Minh chứng:**
![alt text](<image/image copy 14.png>)
### 4. Luồng CI/CD tự động quét lỗ hổng bằng Trivy
*   **Cách chụp:** Vào mục **Actions** trên GitHub, chọn workflow run gần nhất, bấm xem chi tiết bước `Run Trivy vulnerability scanner`. Chụp bảng báo cáo quét lỗ hổng không có lỗi bảo mật nghiêm trọng (HIGH/CRITICAL) nào bị sót lại và pipeline tiếp tục chạy.
*   **Minh chứng:**
![alt text](<image/image copy 9.png>)

### 6. Ký số ảnh container bằng Cosign
*   **Cách chụp:** Trong log chạy GitHub Actions, mở chi tiết bước `Sign the published image` hiển thị kết quả ký số thành công bằng cặp khóa bảo mật và mật khẩu lưu trữ tại GitHub Secrets.
*   **Minh chứng:**
![alt text](<image/image copy 15.png>)


### 8. Email cảnh báo Alertmanager gửi về hộp thư thành công
*   **Cách chụp:** Mở hộp thư Gmail cá nhân dùng để nhận cảnh báo, chụp email nhận được có tiêu đề gửi từ Alertmanager (chẳng hạn như cảnh báo thử nghiệm `TestAlertmanagerEmail`), hiển thị rõ thông tin nội dung cảnh báo.
*   **Minh chứng:**
![alt text](<image/image copy 18.png>)
### 9. OPA Gatekeeper chặn Pod sử dụng tag ":latest"
*   **Cách chụp:** Mở terminal chạy lệnh deploy một Pod sử dụng ảnh có tag `:latest` (ví dụ `nginx:latest`). Chụp màn hình terminal hiển thị thông báo lỗi từ chối từ Gatekeeper (`Admission webhook denied the request: latest tag is not allowed`).
*   **Minh chứng:**
![alt text](<image/image copy 17.png>)
### 10. OPA Gatekeeper chặn Pod bật "hostNetwork: true"
*   **Cách chụp:** Chạy thử một file YAML của Pod cấu hình `hostNetwork: true`. Chụp màn hình thông báo lỗi chặn của Gatekeeper (`The specified hostNetwork and hostPort are not allowed`).
*   **Minh chứng:**
![alt text](<image/image copy 3.png>)
### 11. OPA Gatekeeper chặn Pod không khai báo Resource Limits
*   **Cách chụp:** Thử nghiệm tạo Pod không có phần định nghĩa `limits` hoặc `requests` cho tài nguyên (CPU, Memory). Chụp màn hình thông báo lỗi chặn (`container does not have limits defined`).
*   **Minh chứng:**
![alt text](<image/image copy 16.png>)
### 12. OPA Gatekeeper chặn Deployment vượt giới hạn replicas
*   **Cách chụp:** Thay đổi số lượng replicas trong Deployment vượt quá số lượng tối đa cho phép (ví dụ đặt 5 replicas). Chụp màn hình thông báo lỗi chặn (`The replica count exceeds the maximum allowed`).
*   **Minh chứng:**
![alt text](<image/image copy 4.png>)
---

## 🛠️ PHẦN 2: THÔNG TIN CẤU HÌNH HỆ THỐNG CHÍNH

### Hệ thống Quản trị GitOps (ArgoCD)
*   **Root Application:** `argocd/root.yaml` trỏ tới tất cả các tài nguyên cần thiết.
*   **Thứ tự deploy Policies:** 
    *   `ConstraintTemplates` (Sync Wave 1)
    *   `Constraints` (Sync Wave 2)

### Kênh Cảnh báo & Đồng bộ Mật khẩu
*   **AWS Secrets Manager:** Lưu trữ Gmail App Password tại key `email-password`.
*   **ESO (External Secrets Operator):** Tự động kéo mật khẩu từ AWS mỗi `10 giây` lưu vào K8s Secret `alertmanager-email` trong namespace `monitoring`.
*   **Alertmanager:** Sử dụng mật khẩu trong Secret trên để gửi email cảnh báo thông qua SMTP của Gmail (`smtp.gmail.com:587`).
