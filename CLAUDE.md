# CLAUDE.md — Data Labeling Web App

## Tài liệu tham chiếu

- **Kế hoạch triển khai (Phase roadmap, schema, API):** `.CLAUDE/plan.md`
- **Quy tắc code bắt buộc:** `.CLAUDE/RULE.md`
- **Quy trình lặp lại (migrations, router, annotation type...):** `.CLAUDE/SKILL.md`
- **Session log (việc đã làm theo từng buổi):** `.CLAUDE/MEMORY.md`

Đọc 4 file trên trước khi làm bất kỳ tác vụ nào trong dự án. Đặc biệt đọc `MEMORY.md` để biết trạng thái hiện tại và tránh làm lại việc đã hoàn thành.

---

## Tech Stack

| Layer | Công nghệ |
|---|---|
| Backend | FastAPI (async), SQLAlchemy async + asyncpg, Alembic |
| Frontend | Next.js 14 (App Router, TypeScript strict), Tailwind CSS, shadcn/ui |
| DB | PostgreSQL |
| Canvas | Konva.js + react-konva |
| State | Zustand (annotation session + undo stack) |
| Auth | JWT — access token (header) + refresh token (httpOnly cookie) |
| Storage | Local filesystem, abstracted để swap sang S3 |
| Infrastructure | Docker Compose — toàn bộ stack chạy trong container |

---

## Cấu trúc thư mục

```
labelling/
├── CLAUDE.md                        ← file này
├── docker-compose.yml               ← 3 services: db, backend, frontend
├── .env.example                     ← template biến môi trường
├── .env                             ← giá trị thực (git-ignored)
├── backend/
│   ├── Dockerfile
│   ├── .dockerignore
│   ├── pyproject.toml
│   ├── alembic/
│   └── app/
│       ├── main.py
│       ├── config.py
│       ├── database.py
│       ├── deps.py                  ← RBAC dependency injection
│       ├── models/                  ← SQLAlchemy ORM
│       ├── schemas/                 ← Pydantic request/response
│       ├── routers/                 ← FastAPI APIRouter
│       ├── services/
│       │   ├── storage_service.py   ← local/S3 abstraction + generate_access_token()
│       │   ├── export_service.py    ← COCO, YOLO, CSV, JSON
│       │   └── history_service.py   ← _save_history() trước mọi UPDATE annotation
│       └── utils/
├── frontend/
│   ├── Dockerfile
│   ├── .dockerignore
│   ├── package.json
│   ├── next.config.js
│   ├── tsconfig.json
│   └── src/
│       ├── app/                     ← Next.js App Router pages
│       ├── components/
│       │   ├── labeling/            ← ImageCanvas.tsx (Konva 3-layer)
│       │   ├── projects/
│       │   ├── tasks/
│       │   └── review/
│       ├── store/
│       │   └── annotationStore.ts   ← Zustand (undo stack, tool state)
│       ├── hooks/
│       │   └── useFileUrl.ts        ← hook lấy pre-signed URL, không dùng path trực tiếp
│       └── lib/api.ts               ← Axios + JWT auto-refresh
└── storage/                         ← volume mount file upload (git-ignored)
```

---

## Quyết định kiến trúc không được thay đổi

### 1. JSONB polymorphic cho annotation_data
Bảng `annotations` dùng 1 cột JSONB thay vì 4 bảng riêng cho 4 loại annotation. Cho phép thêm loại mới (keypoint, 3D box...) mà không cần migration schema.

### 2. File serving qua pre-signed URL
Không bao giờ expose `file_path` ra API response. Mọi URL file đều đi qua `storage_service.generate_access_token()` → `GET /files/{token}`. Pattern này giữ nguyên khi swap Local → S3.

### 3. Konva ImageCanvas — 3 layer cố định
```
Layer 1 (bottom)  KonvaImage          ảnh tĩnh, listening=false
Layer 2 (middle)  committed shapes    redraw khi danh sách thay đổi
Layer 3 (top)     active tool         redraw liên tục theo mouse
```
Không gộp tất cả vào 1 layer — gây drop FPS với 50+ annotation.

### 4. Optimistic locking trên annotations
Cột `version INTEGER` trên bảng `annotations`. `PATCH /annotations/{id}` phải nhận và kiểm tra `version` từ client, trả `409` nếu stale.

### 5. Audit trail bắt buộc
SQLAlchemy `before_update` Event Listener trên model `Annotation` tự động gọi `history_service` — không gọi thủ công trong service, không có exception về human error.

---

## Files bắt đầu từ đây (theo thứ tự)

1. `backend/app/models/annotation.py` — JSONB polymorphic + version column
2. `backend/app/deps.py` — RBAC dependency injection
3. `backend/app/services/storage_service.py` — abstraction layer + generate_access_token()
4. `backend/app/routers/files.py` — serve file qua token
5. `frontend/src/store/annotationStore.ts` — Zustand store
6. `frontend/src/components/labeling/ImageCanvas.tsx` — Konva 3-layer
7. `backend/app/services/export_service.py` — COCO/YOLO/CSV/JSON
8. `backend/app/services/history_service.py` — audit trail, triggered tự động qua SQLAlchemy `before_update` Event Listener

---

## Lệnh thường dùng

```bash
# Khởi động toàn bộ stack (xem chi tiết trong SKILL.md: dev-start)
docker compose up

# Migration (chạy trong container)
docker compose exec backend alembic revision --autogenerate -m "<mô tả>"
docker compose exec backend alembic upgrade head

# Seed dữ liệu test
docker compose exec backend python -m app.utils.seed

# Kiểm tra health
curl http://localhost:8000/health
```

---

## Phase hiện tại

Xem `plan.md` để biết Phase nào đang triển khai và Deliverable cần đạt. Không implement Phase N+1 khi Phase N chưa pass verification.
