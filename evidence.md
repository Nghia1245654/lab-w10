ý do mà quy tắc chưa chặn pod vi phạm là vì lỗi đồng bộ tuần tự của ArgoCD (CRD Bootstrapping Race):

Khi chúng ta gộp chung cả ConstraintTemplates (để tạo ra các CRD định nghĩa luật) và Constraints (khởi tạo luật thực tế) vào cùng một ứng dụng ArgoCD (gatekeeper-policies), ArgoCD sẽ thực hiện kiểm tra kiểm thử (dry-run validation) tất cả các file cùng một lúc.

Do CRD từ các ConstraintTemplates chưa được Kubernetes API server tạo xong, API server đã từ chối validate các file Constraints (báo lỗi không tìm thấy CRD).
Việc này làm cho toàn bộ tiến trình đồng bộ của ArgoCD đối với ứng dụng này bị SyncFailed và không có luật chặn nào được kích hoạt.