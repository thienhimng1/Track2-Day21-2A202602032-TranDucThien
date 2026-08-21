# BÁO CÁO THỰC HÀNH MLOPS: TỪ THỰC NGHIỆM CỤC BỘ ĐẾN TRIỂN KHAI LIÊN TỤC

**Khóa học**: AIInAction - VinUni | **Buổi**: Day 21 - CI/CD cho AI Systems  
**Học viên**: Trần Đức Thiện  
**GitHub Repository**: [https://github.com/thienhimng1/Track2-Day21-2A202602032-TranDucThien](https://github.com/thienhimng1/Track2-Day21-2A202602032-TranDucThien)  

---

## 1. Danh Sách Ảnh Minh Chứng (Screenshots)

1. `01_mlflow_experiments.png`: Giao diện MLflow UI hiển thị các lần chạy thực nghiệm và so sánh chỉ số.
2. `02_github_actions_pipeline.png`: Tab GitHub Actions hiển thị pipeline `MLOps Pipeline CICD` chạy thành công cả 4 jobs (`Test`, `Train`, `Eval`, `Deploy`).
3. `03_fastapi_inference_results.png`: Terminal kiểm tra endpoint `GET /health` (`status: ok`) và `POST /predict` (`prediction: 2, label: cao`) trên Cloud VM.
4. `04_gcs_cloud_storage.png`: Google Cloud Storage Console hiển thị thư mục `dvc/` và `models/latest/model.pkl`.

---

## 2. Kết Quả Thực Nghiệm & Lựa Chọn Siêu Tham Số (Bước 1)

Trên tập dữ liệu ban đầu `train_phase1.csv` (2998 mẫu) và tập đánh giá `eval.csv` (500 mẫu), các thí nghiệm huấn luyện mô hình `RandomForestClassifier` được theo dõi bằng MLflow với kết quả:

| Thí nghiệm | Siêu tham số (`params.yaml`) | Accuracy | F1-Score |
| :--- | :--- | :---: | :---: |
| Run 1 | `n_estimators: 50, max_depth: 3, min_samples_split: 2` | 0.5580 | 0.5185 |
| Run 2 | `n_estimators: 100, max_depth: 5, min_samples_split: 2` | 0.5640 | 0.5534 |
| Run 3 | `n_estimators: 200, max_depth: 10, min_samples_split: 5` | 0.6440 | 0.6417 |
| **Run 4 (Tối ưu)** | **`n_estimators: 300, max_depth: 25, min_samples_split: 2`** | **0.6760** | **0.6751** |

- **Lý do chọn bộ tham số**: `n_estimators: 300` cùng `max_depth: 25` giúp mô hình học được các ranh giới phi tuyến tính phức tạp giữa các đặc trưng hóa học của rượu vang mà không bị underfitting như các cây có độ sâu thấp (`max_depth: 3, 5`), đạt độ chính xác cao nhất `0.6760`.

---

## 3. Kiến Trúc Pipeline CI/CD & Continuous Training (Bước 2 & Bước 3)

- **Bước 2 (Kiểm soát chất lượng với Eval Gate)**:
  - Khi huấn luyện trên `train_phase1.csv`, mô hình đạt `Accuracy = 0.6760 < 0.70`.
  - **Eval Gate** đã kích hoạt chính xác theo thiết kế, chặn job `Deploy` để ngăn triển khai mô hình chưa đạt chuẩn lên môi trường Production.
- **Bước 3 (Huấn luyện liên tục khi có dữ liệu mới)**:
  - Bổ sung `train_phase2.csv` nâng tổng tập huấn luyện lên 5996 mẫu.
  - Phiên bản hóa dữ liệu qua DVC và đẩy commit lên GitHub.
  - Pipeline GitHub Actions tự động kích hoạt hoàn toàn: `Accuracy` tăng vọt lên **0.7580 (75.8% > 0.70)**.
  - **Eval Gate vượt qua thành công** ➔ Job `Deploy` tự động SSH vào Cloud VM (GCP GCE `34.44.82.252`) và khởi động lại dịch vụ FastAPI.

---

## 4. Khó Khăn Gặp Phải & Giải Pháp Xử Lý

1. **Chính sách bảo mật GCP chặn tạo Service Account Key (`constraints/iam.disableServiceAccountKeyCreation`)**:
   - *Khó khăn*: GCP tự động bật chính sách "Secure by Default" ở cấp Organization, không cho phép tải file JSON key.
   - *Giải pháp*: Truy cập Organization Policies trên GCP Console, cấu hình `Override parent's policy` và chuyển trạng thái `Enforcement` thành `Off` để cấp quyền tạo `sa-key.json`.
2. **Quyền hạn DVC trên Google Cloud Storage**:
   - *Khó khăn*: Lần đầu chạy `dvc pull` bị lỗi `Anonymous caller / 401 Unauthorized` do thiếu liên kết Service Account.
   - *Giải pháp*: Gán role `roles/storage.objectAdmin` cho Service Account trên bucket và cấu hình biến môi trường `GOOGLE_APPLICATION_CREDENTIALS` cùng `dvc config credentialpath` trong pipeline CI/CD.
3. **Cú pháp gửi JSON trên Windows PowerShell**:
   - *Khó khăn*: `curl.exe` trên PowerShell bị lỗi escape ký tự `\"` trong chuỗi JSON.
   - *Giải pháp*: Sử dụng cmdlet chuẩn `Invoke-RestMethod` của PowerShell với `-ContentType "application/json"`.
