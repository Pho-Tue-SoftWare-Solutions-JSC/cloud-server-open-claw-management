# Changelog — Repo Reference Update (2026-03-26)

## Mục tiêu
- Đồng bộ toàn bộ tài liệu và script cài đặt/cập nhật về repo hiện tại:
  - `Pho-Tue-SoftWare-Solutions-JSC/cloud-server-open-claw-management`
  - nhánh `main`

## File đã cập nhật
- `README.md`
- `bootstrap.sh`
- `Architecture.md`
- `docs/update-guide.md`

## Nội dung thay đổi
- Chuẩn hóa các URL `raw.githubusercontent.com` đang trỏ sang repo/nhánh cũ.
- Đổi các tham chiếu từ:
  - `Pho-Tue-SoftWare-Solutions-JSC/vps-openclaw-management`
  - nhánh `v2` hoặc `main` cũ liên quan
- Sang:
  - `Pho-Tue-SoftWare-Solutions-JSC/cloud-server-open-claw-management`
  - nhánh `main`

## Chi tiết
### `README.md`
- Giữ lệnh cài đặt nhanh trỏ đúng về `cloud-server-open-claw-management/main/install.sh`.

### `bootstrap.sh`
- Xác nhận `REPO_RAW` dùng đúng repo `cloud-server-open-claw-management/main`.

### `Architecture.md`
- Cập nhật lệnh cài đặt nhanh sang repo hiện tại.
- Cập nhật lệnh tải `management-api/server.js` thủ công sang repo hiện tại.

### `docs/update-guide.md`
- Cập nhật `REPO_RAW` sang repo hiện tại để đồng bộ quy trình update thủ công.

## Ghi chú
- Các thay đổi chỉ tập trung vào tham chiếu repo/nhánh.
- Không thay đổi logic triển khai ngoài phần đã chỉnh trước đó trong `install.sh` và `management-api/server.js`.
- Sau rà soát, các file mục tiêu không còn tham chiếu `vps-openclaw-management/v2`.

## Trạng thái
- Hoàn tất.
- Đã kiểm tra lại và không còn tham chiếu repo cũ trong các file mục tiêu.
