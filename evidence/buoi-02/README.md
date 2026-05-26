# Evidence Buổi 02

Thư mục này lưu bằng chứng Lab 02.

Cần có:

```text
evidence/buoi-02/
  README.md
  checklist.md
  known-issues.md
  spectral-report.txt
  tool-versions.txt
  git-log.txt
  mock-screenshots/
    req-01-*.png
    req-02-*.png
    req-03-*.png
    req-04-*.png
    req-05-*.png
```

## Cách sinh report Spectral

```bash
./scripts/collect_session02_evidence.sh
```

Windows:

```powershell
.\scripts\collect_session02_evidence.ps1
```

## Ảnh mock server

Lab 02 chưa yêu cầu Postman. Minh chứng nên là ảnh chụp Terminal/PowerShell khi chạy `curl` tới Prism mock server.

Mỗi ảnh cần thể hiện:

- lệnh `curl` trong Terminal/PowerShell;
- status code;
- response body;
- URL `http://localhost:4010/...`.
