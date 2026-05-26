# Checklist Lab 02

- [ ] Đã xác định đúng cặp đàm phán.
- [ ] Đã đọc đúng user story trong thư mục `user-stories/`.
- [ ] Provider đã điền `docs/analysis-provider.md`.
- [ ] Consumer đã điền `docs/analysis-consumer.md`.
- [ ] `openapi.yaml` khai báo `openapi: 3.1.0`.
- [ ] Có tối thiểu 4 path.
- [ ] Mỗi operation có `operationId`, `summary`, `description`, `tags`.
- [ ] Schema lớn đã đưa vào `components/schemas`.
- [ ] Có `oneOf` + `discriminator`.
- [ ] Có union type với `null`, không dùng `nullable: true`.
- [ ] Có `Problem` schema cho response lỗi.
- [ ] `spectral lint` không có severity error.
- [ ] Đã lưu `evidence/buoi-02/spectral-report.txt`.
- [ ] Prism mock server chạy được ở port 4010.
- [ ] Có 5 ảnh request mẫu trong `mock-screenshots/`.
- [ ] `negotiation-log.md` có tối thiểu 6 issue.
- [ ] Có sign-off Provider, Consumer, Witness.
- [ ] Đã hoàn thiện `VERSIONING.md` cho bài tập về nhà.
