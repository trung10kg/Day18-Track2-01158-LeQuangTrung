# Reflection

**Anti-pattern rủi ro nhất: tin job bảo trì tự động mà không đo lại kết quả.**

NB6 đo hai bug cụ thể. `VACUUM` retention 7 ngày không xoá được 3 file
orphan do job crash để lại, vì `deltalake` chỉ thu hồi file *đã tombstone
trong log* — file crash không bao giờ vào log nên vô hình ở mọi retention.
`expire_snapshots` đưa Iceberg từ 20 xuống 3 snapshot nhưng xoá **0 file
avro**: job chỉ dọn metadata, data file nằm nguyên tới khi chạy orphan-sweep.

Team chúng tôi lưu trajectory + embedding cho agent (NB7/NB8), ghi liên tục
qua re-run, retry và backfill trùng ngày — đúng điều kiện sinh orphan: job
retry ghi file mới trước khi commit, job cũ crash giữa chừng. Nếu chỉ lên
lịch `VACUUM`/`expire_snapshots` và tin "log SUCCESS là đã dọn xong", chúng
tôi sẽ tích orphan vô hình — hoá đơn lưu trữ tăng dần mà không job nào báo
lỗi, đúng triệu chứng NB6: "đã expire mà hoá đơn S3 không giảm".

Khắc phục: luôn chạy Job 3 và Job 4 thành **cặp bắt buộc**, đo dung lượng
đĩa trước–sau thay vì tin log job.
