[![Review Assignment Due Date](https://classroom.github.com/assets/deadline-readme-button-22041afd0340ce965d47ae6ef1cefeee28c7c493a6346c4f15d667ab976d596c.svg)](https://classroom.github.com/a/Iwvz6vup)
# FIT4110 — Lab 02 OpenAPI 3.1 Contract-First

Repo này dùng cho **Lab 02 — Thực hành đàm phán và viết OpenAPI 3.1** của học phần **Dịch vụ kết nối và công nghệ nền tảng (FIT4110)**.

Lab 02 nối tiếp trực tiếp Lab 01 trong repo `FIT4110_setup`:

- Lab 01: sinh viên thiết lập môi trường và nộp `service-boundary.md`.
- Lab 02: sinh viên chuyển Service Boundary thành **hợp đồng API** bằng `openapi.yaml`.
- Hai nhóm ở hai vai **Consumer** và **Provider** phải đàm phán trước khi code.
- Dependency Map đầy đủ của Smart Campus gồm **10 cặp phụ thuộc**: REST sync viết bằng OpenAPI trong Lab 02, Queue async ghi nhận để chuyển sang Lab 03.

> Nguyên tắc trọng tâm: **contract-first** — không viết code trước khi chốt hợp đồng API.

---

## 1. Sinh viên cần làm gì trong Lab 02?

Mỗi cặp đàm phán cần hoàn thành các artefact sau:

```text
openapi.yaml
negotiation-log.md
docs/analysis-provider.md
docs/analysis-consumer.md
evidence/buoi-02/spectral-report.txt
evidence/buoi-02/mock-screenshots/req-01-*.png ... req-05-*.png
VERSIONING.md
```

Bài làm đạt yêu cầu khi:

- `openapi.yaml` dùng **OpenAPI 3.1.0**.
- Có tối thiểu 4 path phù hợp user story của cặp.
- Có schema đặt trong `components/schemas`, dùng `$ref` thay vì inline schema dài.
- Có ít nhất một ví dụ `oneOf` + `discriminator`.
- Có ít nhất một trường dùng union type với `null`, ví dụ `type: [string, "null"]`.
- Response lỗi 4xx/5xx dùng `Problem Details`.
- File pass `spectral lint` với `campus-spectral.yaml`.
- Mock server chạy được bằng Prism và có 5 request mẫu làm bằng chứng.
- `negotiation-log.md` có tối thiểu 6 issue, rationale rõ, có sign-off 2 bên.

---

## 2. Cấu trúc repo

```text
FIT4110_lab02_openapi/
  README.md
  openapi.yaml
  campus-spectral.yaml
  negotiation-log.md
  package.json
  docs/
    lab02-guide.md
    pairing-matrix.md
    openapi-authoring-guide.md
    swagger-online-workflow.md
    event-contract-template.md
    analysis-provider.md
    analysis-consumer.md
  user-stories/
    README.md
    pair-01-camera-ai-vision.md
    pair-02-core-ai-vision.md
    pair-03-core-access-gate.md
    pair-04-core-notification-async.md
    pair-05-iot-core-async.md
    pair-06-iot-analytics-async.md
    pair-07-camera-analytics-async.md
    pair-08-core-analytics-async.md
    pair-09-access-analytics-async.md
    pair-10-access-core-policy.md
  evidence/
    buoi-02/
      README.md
      checklist.md
      known-issues.md
      mock-screenshots/
        .gitkeep
  scripts/
    install_lab02_cli.sh
    install_lab02_cli.ps1
    lint_openapi.sh
    lint_openapi.ps1
    mock_openapi.sh
    mock_openapi.ps1
    collect_session02_evidence.sh
    collect_session02_evidence.ps1
  .github/workflows/check-lab02.yml
```

---

## 3. Cài đặt công cụ

Yêu cầu tối thiểu:

- Git
- Node.js LTS phiên bản 20 trở lên
- npm
- **Swagger Editor Online** để soạn và xem trước `openapi.yaml`
- VS Code hoặc editor YAML/OpenAPI tương đương nếu muốn sửa offline
- `curl` để gọi thử Prism mock server; 

### macOS/Linux

```bash
chmod +x scripts/*.sh
./scripts/install_lab02_cli.sh
```

### Windows PowerShell

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\scripts\install_lab02_cli.ps1
```

Có thể cài qua npm script:

```bash
npm run install:cli
```

---

## 4. Quy trình thực hành gợi ý

### Bước 1. Xác định cặp đàm phán

Xem bảng trong:

```text
docs/pairing-matrix.md
```

Mở đúng user story trong thư mục:

```text
user-stories/
```

Ví dụ REST sync:

```text
user-stories/pair-01-camera-ai-vision.md
```

Ví dụ Queue async:

```text
user-stories/pair-08-core-analytics-async.md
```

Queue async được đưa vào Dependency Map để không bỏ sót kiến trúc, nhưng Lab 02 chưa yêu cầu Postman và chưa yêu cầu đặc tả AsyncAPI đầy đủ.

---

### Bước 2. Mỗi bên phân tích độc lập

Provider điền:

```text
docs/analysis-provider.md
```

Consumer điền:

```text
docs/analysis-consumer.md
```

Không trao đổi quá sớm. Mục tiêu là mỗi bên phải có góc nhìn riêng để đưa vào bàn đàm phán.

---

### Bước 3. Viết bản thảo `openapi.yaml` bằng Swagger Editor Online

Trong Lab 02, sinh viên ưu tiên dùng **Swagger Editor Online** tại:

```text
https://editor.swagger.io
```

Quy trình gợi ý:

1. Mở `openapi.yaml` trong repo.
2. Copy toàn bộ nội dung YAML.
3. Dán vào khung bên trái của Swagger Editor Online.
4. Quan sát phần preview bên phải để phát hiện lỗi cú pháp, thiếu schema, thiếu response hoặc mô tả chưa rõ.
5. Sau khi chỉnh xong, copy YAML từ Swagger Editor về lại file `openapi.yaml` trong repo.
6. Commit file đã sửa lên GitHub.

> Swagger Editor Online giúp xem nhanh tài liệu API và lỗi cú pháp. Tuy nhiên, tiêu chí chấm chính vẫn là `spectral lint` với `campus-spectral.yaml`, vì Swagger Editor không kiểm tra đầy đủ rule riêng của lớp.

Có thể bắt đầu từ file mẫu trong repo:

```text
openapi.yaml
```

Sau khi sửa, kiểm tra bằng:

```bash
npm run lint
```

hoặc:

```bash
./scripts/lint_openapi.sh
```

Windows:

```powershell
.\scripts\lint_openapi.ps1
```

---

### Bước 4. Đàm phán và ghi biên bản

Hai nhóm cùng mở:

```text
negotiation-log.md
```

Mỗi issue cần có:

- Bối cảnh
- Vấn đề
- Đề xuất
- Quyết định
- Rationale
- Tác động đến service

Commit bản đã ký:

```bash
git add openapi.yaml negotiation-log.md docs/analysis-provider.md docs/analysis-consumer.md
 git commit -m "chore(contract): <ten-cap> v1.0 signed-off"
```

---

### Bước 5. Chạy Spectral và lưu báo cáo

```bash
npm run lint:report
```

hoặc:

```bash
./scripts/collect_session02_evidence.sh
```

Kết quả cần có:

```text
evidence/buoi-02/spectral-report.txt
```

---

### Bước 6. Chạy Prism mock server và test bằng curl

```bash
npm run mock
```

hoặc:

```bash
./scripts/mock_openapi.sh
```

Server mặc định chạy ở:

```text
http://localhost:4010
```

Lab 02 **không yêu cầu dùng Postman**. Sinh viên test 5 request mẫu bằng `curl` trong Terminal/PowerShell hoặc chạy script mẫu:

```bash
./scripts/test_mock_with_curl.sh
```

Windows:

```powershell
.\scripts\test_mock_with_curl.ps1
```

Khi chụp minh chứng, ảnh cần có lệnh `curl`, status code và response body. Lưu ảnh vào:

```text
evidence/buoi-02/mock-screenshots/
```

---

## 5. Lệnh kiểm tra nhanh

```bash
node --version
npm --version
spectral --version
prism --version
spectral lint openapi.yaml --ruleset campus-spectral.yaml
prism mock openapi.yaml --port 4010
```

Ví dụ gọi mock bằng `curl`:

```bash
curl -i http://localhost:4010/health
curl -i http://localhost:4010/alerts/recent -H "Authorization: Bearer test-token"
```

---

## 6. Nộp bài

```bash
git status
git add openapi.yaml negotiation-log.md VERSIONING.md docs evidence/buoi-02
git commit -m "submit: lab02 openapi contract evidence"
git push
```

---

## 7. Không được commit

Không commit các file sau:

```text
*.doc
*.docx
*.ppt
*.pptx
.env
node_modules/
file dữ liệu lớn
file model lớn
```

Repo đã có GitHub Actions để chặn file Word và kiểm tra cấu trúc Lab 02.

---

## 8. Tinh thần của Lab 02

> Không nộp “API em nghĩ là đúng”, mà nộp **hợp đồng API đã được đàm phán, kiểm tra và có bằng chứng chạy được**.
