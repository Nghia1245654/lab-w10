# Danh sách các Lỗi và Khó khăn trong Quá trình Triển khai (Lab W10)

Báo cáo này tổng hợp chi tiết các vấn đề kỹ thuật phát sinh trong quá trình vận hành hệ thống GitOps, CI/CD bảo mật và quản lý chính sách cụm Kubernetes, cùng với giải pháp đã áp dụng thành công.

---

## 1. Lỗi đồng bộ tuần tự của ArgoCD (CRD Bootstrapping Race)

### 🔴 Khó khăn gặp phải
Khi gộp chung cả **ConstraintTemplates** (để tạo ra các CRD định nghĩa luật) và **Constraints** (khởi tạo luật thực tế) vào cùng một ứng dụng ArgoCD (`gatekeeper-policies`), ArgoCD sẽ thực hiện kiểm tra kiểm thử (dry-run validation) tất cả các file cùng một lúc.
*   Do CRD từ các ConstraintTemplates chưa được Kubernetes API server tạo xong, API server đã từ chối validate các file Constraints (báo lỗi không tìm thấy CRD).
*   Việc này làm cho toàn bộ tiến trình đồng bộ của ArgoCD đối với ứng dụng này bị `SyncFailed` và không có luật chặn nào được kích hoạt.

### 🟢 Giải pháp khắc phục
Áp dụng cơ chế **ArgoCD Sync Waves** để phân chia thứ tự triển khai tài nguyên theo các bước tuần tự:
*   Thêm annotation `argocd.argoproj.io/sync-wave: "1"` vào tất cả các **ConstraintTemplates** để ép chúng tạo trước.
*   Thêm annotation `argocd.argoproj.io/sync-wave: "2"` vào tất cả các **Constraints** tương ứng để đảm bảo chúng chỉ được kiểm duyệt và tạo sau khi các CRD đã sẵn sàng dưới cụm.

---

## 2. Gmail SMTP bị chặn do Spam Cảnh báo Hệ thống (Gmail Rate-limiting)

### 🔴 Khó khăn gặp phải
Sau khi cấu hình Alertmanager gửi email cảnh báo về hòm thư Gmail cá nhân, cụm local (Minikube/Kind) liên tục sinh ra các cảnh báo lỗi hệ thống giả lập như `KubeControllerManagerDown`, `KubeSchedulerDown`, `etcdMembersDown` (do cụm local chặn không cho Prometheus quét các cổng nội bộ, chứ bản thân cụm không bị lỗi).
*   Alertmanager bị "ngập" trong hàng trăm cảnh báo rác và gửi liên tục về Gmail.
*   Hệ thống bảo mật của Google phát hiện hành vi đáng ngờ và tạm thời khóa IP/tài khoản gửi thư với mã lỗi: `454 4.7.0 Too many login attempts`.

### 🟢 Giải pháp khắc phục
*   Tối ưu hóa cấu hình Helm values của `kube-prometheus-stack` để tắt hoàn toàn tính năng quét giám sát các thành phần hệ thống nội bộ local (`kubeControllerManager`, `kubeScheduler`, `kubeEtcd`, `kubeProxy`) vốn không cần thiết trên môi trường phát triển local.
*   Cấu hình loại bỏ cảnh báo nhịp tim `Watchdog` bằng cách chuyển hướng nó sang `null` receiver.
*   Khôi phục luồng gửi email mặc định (`receiver: email-notifications`) để đảm bảo hệ thống vẫn gửi mail báo về bình thường khi phát sinh lỗi thực tế (như lỗi ứng dụng, quá tải tài nguyên), nhưng hòm thư không còn bị spam rác.

---

## 3. Đồng bộ mật khẩu tự động bằng External Secrets Operator (ESO) và AWS Secrets Manager

### 🔴 Khó khăn gặp phải
Khi cố gắng kiểm thử việc đổi mật khẩu, nếu người dùng chỉnh sửa giá trị thủ công ở máy local (trong K8s Secret `alertmanager-email`), thì chỉ sau 10 giây giá trị đó sẽ bị ghi đè ngược trở lại về mật khẩu cũ. Điều này gây khó hiểu về cơ chế hoạt động của Controller.

### 🟢 Giải pháp khắc phục
*   Làm rõ cơ chế hoạt động **Reconciliation Loop (Vòng lặp đồng hòa)** của Kubernetes.
*   Xác định rõ **AWS Secrets Manager là Single Source of Truth** (nguồn gốc thông tin duy nhất). Mọi hành động cập nhật mật khẩu bắt buộc phải thực hiện trên đám mây AWS Secrets Manager. Hệ thống ESO ở local sẽ tự động phát hiện thay đổi và đồng bộ xuống cụm K8s sau mỗi `10s` (theo tham số `refreshInterval: 10s`) mà không cần can thiệp thủ công.

---

## 4. Trivy Quét Lỗi và Chặn đứng Quy trình CI/CD

### 🔴 Khó khăn gặp phải
Ban đầu, Dockerfile của ứng dụng sử dụng các base image cũ hoặc chứa nhiều thư viện hệ thống lỗi thời. Khi pipeline GitHub Actions chạy, bước quét lỗ hổng bảo mật bằng **Trivy** phát hiện nhiều lỗi bảo mật nghiêm trọng (HIGH/CRITICAL) và lập tức dừng tiến trình build (`exit-code: 1`), khiến ảnh không thể được đẩy lên GHCR để triển khai.

### 🟢 Giải pháp khắc phục
*   Tối ưu hóa lại `Dockerfile` của ứng dụng API.
*   Chuyển sang sử dụng base image sạch, tối giản và bảo mật hơn là **`python:3.13-alpine`**.
*   Loại bỏ các gói thư viện không cần thiết nhằm thu hẹp diện tích tấn công (attack surface), giúp vượt qua khâu kiểm duyệt ngặt nghèo của Trivy mà vẫn đảm bảo ứng dụng hoạt động ổn định.