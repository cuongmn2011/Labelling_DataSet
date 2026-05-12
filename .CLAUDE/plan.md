# Data Labeling Web App - Kế hoạch triển khai

## Context
Xây dựng một web app labeling dữ liệu từ đầu (greenfield) để tạo training dataset cho ML/AI. Tool cần hỗ trợ 4 loại annotation, quản lý nhiều project độc lập, nhiều người dùng cùng label, và export ra các format phổ biến.

---

## Tech Stack
- **Frontend**: Next.js 14 (App Router, TypeScript strict), Tailwind CSS, shadcn/ui
- **Backend**: FastAPI (Python, async), SQLAlchemy async + asyncpg, Alembic
- **DB**: PostgreSQL
- **Storage**: Local filesystem (abstracted để swap sang S3 sau)
- **Auth**: JWT (access + refresh token, httpOnly cookie)
- **Infrastructure**: Docker Compose — toàn bộ stack (db, backend, frontend) chạy trong container

---

## Cấu trúc thư mục

```
labelling/
├── docker-compose.yml            ← 3 services: db, backend, frontend
├── .env.example                  ← template biến môi trường
├── .env                          ← giá trị thực (git-ignored)
├── backend/
│   ├── Dockerfile
│   ├── .dockerignore
│   ├── pyproject.toml
│   ├── alembic/
│   └── app/
│       ├── main.py
│       ├── config.py
│       ├── database.py
│       ├── deps.py               ← RBAC dependency injection
│       ├── models/               ← SQLAlchemy ORM
│       ├── schemas/              ← Pydantic request/response
│       ├── routers/              ← FastAPI APIRouter
│       ├── services/
│       │   ├── storage_service.py   ← local/S3 abstraction
│       │   └── export_service.py    ← COCO, YOLO, CSV, JSON
│       └── utils/
├── frontend/
│   ├── Dockerfile
│   ├── .dockerignore
│   ├── package.json
│   ├── next.config.js
│   ├── tsconfig.json
│   └── src/
│       ├── app/                  ← Next.js App Router pages
│       ├── components/
│       │   ├── labeling/         ← Canvas tools (Konva.js)
│       │   ├── projects/
│       │   ├── tasks/
│       │   └── review/
│       ├── store/
│       │   └── annotationStore.ts   ← Zustand (undo stack, tool state)
│       ├── hooks/
│       └── lib/api.ts            ← Axios + JWT auto-refresh
└── storage/                      ← volume mount cho file upload (git-ignored)
```

---

## Database Schema (PostgreSQL)

### Sơ đồ quan hệ

```
users ──< project_members >── projects ──< label_classes
                                  │
                                  └──< data_items ──< annotations >── label_classes
                                  │         │               │
                                  └──< tasks ┘         created_by (users)
                                        │
                                        └──< reviews
                                  │
                                  └──< export_jobs
```

---

### Bảng `users`
```sql
id               UUID PRIMARY KEY DEFAULT gen_random_uuid()
email            VARCHAR(255) UNIQUE NOT NULL
username         VARCHAR(100) UNIQUE NOT NULL
hashed_password  VARCHAR(255) NOT NULL
role             VARCHAR(20) NOT NULL DEFAULT 'labeler'
                 -- 'admin' | 'labeler' | 'reviewer'
is_active        BOOLEAN DEFAULT TRUE
created_at       TIMESTAMPTZ DEFAULT NOW()
updated_at       TIMESTAMPTZ DEFAULT NOW()
```

---

### Bảng `projects`
```sql
id           UUID PRIMARY KEY DEFAULT gen_random_uuid()
name         VARCHAR(255) NOT NULL
description  TEXT
type         VARCHAR(30) NOT NULL
             -- 'image_classification' | 'object_detection'
             -- 'image_segmentation'   | 'text_classification'
status       VARCHAR(20) DEFAULT 'active'   -- 'active' | 'archived'
owner_id     UUID REFERENCES users(id)
settings     JSONB DEFAULT '{}'
             -- {"min_labelers": 1, "allow_skip": true, "consensus_threshold": 0.8}
created_at   TIMESTAMPTZ DEFAULT NOW()
updated_at   TIMESTAMPTZ DEFAULT NOW()
deleted_at   TIMESTAMPTZ
```

---

### Bảng `project_members`
```sql
id          UUID PRIMARY KEY DEFAULT gen_random_uuid()
project_id  UUID REFERENCES projects(id) ON DELETE CASCADE
user_id     UUID REFERENCES users(id) ON DELETE CASCADE
role        VARCHAR(20) NOT NULL
            -- 'manager' | 'labeler' | 'reviewer'
joined_at   TIMESTAMPTZ DEFAULT NOW()

UNIQUE(project_id, user_id)
```

---

### Bảng `label_classes`
```sql
id             UUID PRIMARY KEY DEFAULT gen_random_uuid()
project_id     UUID REFERENCES projects(id) ON DELETE CASCADE
name           VARCHAR(100) NOT NULL
color          VARCHAR(7) NOT NULL      -- hex: "#FF5733"
shortcut_key   VARCHAR(5)              -- "1", "q", "F1"...
attributes     JSONB DEFAULT '[]'
               -- [{"name":"truncated","type":"bool"},
               --  {"name":"difficulty","type":"enum","options":["easy","hard"]}]
display_order  INTEGER DEFAULT 0
created_at     TIMESTAMPTZ DEFAULT NOW()

UNIQUE(project_id, name)
```

---

### Bảng `data_items`
```sql
id            UUID PRIMARY KEY DEFAULT gen_random_uuid()
project_id    UUID REFERENCES projects(id) ON DELETE CASCADE
type          VARCHAR(10) NOT NULL     -- 'image' | 'text'

-- Chỉ dùng khi type = 'image'
file_path     VARCHAR(500)            -- đường dẫn tương đối trong storage
file_name     VARCHAR(255)
mime_type     VARCHAR(100)            -- "image/jpeg", "image/png"
width         INTEGER
height        INTEGER

-- Chỉ dùng khi type = 'text'
text_content  TEXT

-- Chung
metadata      JSONB DEFAULT '{}'      -- tên file gốc, tags, nguồn...
status        VARCHAR(20) DEFAULT 'pending'
-- INDEX USING GIN (metadata) — xem Phase 1 migration notes
              -- 'pending' | 'in_progress' | 'labeled' | 'reviewed' | 'skipped'
created_at    TIMESTAMPTZ DEFAULT NOW()
deleted_at    TIMESTAMPTZ

INDEX(project_id, status)
```

---

### Bảng `annotations`  ← quan trọng nhất
```sql
id              UUID PRIMARY KEY DEFAULT gen_random_uuid()
data_item_id    UUID REFERENCES data_items(id) ON DELETE CASCADE
label_class_id  UUID REFERENCES label_classes(id) ON DELETE SET NULL
created_by      UUID REFERENCES users(id)
task_id         UUID REFERENCES tasks(id)

-- JSONB polymorphic theo loại project:
--
-- image_classification:
--   {"confirmed": true}
--   (label_class_id đã là đủ thông tin)
--
-- object_detection:
--   {"x": 10, "y": 20, "width": 150, "height": 80, "rotation": 0}
--   (tọa độ pixel tuyệt đối)
--
-- image_segmentation:
--   {"points": [[10,20],[50,80],[30,120]], "is_crowd": false}
--   (danh sách điểm của polygon)
--
-- text_classification (document-level):
--   {"confirmed": true}
--
-- text_classification (span-level):
--   {"start_offset": 12, "end_offset": 45, "span_text": "Hà Nội"}
annotation_data  JSONB NOT NULL

version      INTEGER DEFAULT 1 NOT NULL  -- optimistic locking: client gửi kèm khi PATCH
is_flagged   BOOLEAN DEFAULT FALSE       -- reviewer đánh dấu annotation xấu
created_at   TIMESTAMPTZ DEFAULT NOW()
updated_at   TIMESTAMPTZ DEFAULT NOW()
deleted_at   TIMESTAMPTZ

INDEX(data_item_id) WHERE deleted_at IS NULL
INDEX USING GIN (annotation_data)       -- tránh Full Table Scan khi Export filter JSONB
```

---

### Bảng `tasks`
```sql
id            UUID PRIMARY KEY DEFAULT gen_random_uuid()
project_id    UUID REFERENCES projects(id) ON DELETE CASCADE
data_item_id  UUID REFERENCES data_items(id) ON DELETE CASCADE
assigned_to   UUID REFERENCES users(id)
assigned_by   UUID REFERENCES users(id)
status        VARCHAR(20) DEFAULT 'assigned'
              -- 'assigned' | 'in_progress' | 'submitted' | 'approved' | 'rejected'
due_date      TIMESTAMPTZ
started_at    TIMESTAMPTZ
submitted_at  TIMESTAMPTZ
created_at    TIMESTAMPTZ DEFAULT NOW()
updated_at    TIMESTAMPTZ DEFAULT NOW()

UNIQUE(data_item_id, assigned_to)   -- 1 labeler chỉ được assign 1 lần/item
INDEX(assigned_to, status)
INDEX(project_id, status)
```

---

### Bảng `reviews`
```sql
id           UUID PRIMARY KEY DEFAULT gen_random_uuid()
task_id      UUID REFERENCES tasks(id) ON DELETE CASCADE
reviewer_id  UUID REFERENCES users(id)
status       VARCHAR(20) DEFAULT 'pending'
             -- 'pending' | 'approved' | 'rejected' | 'needs_revision'
feedback     TEXT                    -- comment của reviewer
reviewed_at  TIMESTAMPTZ
created_at   TIMESTAMPTZ DEFAULT NOW()
updated_at   TIMESTAMPTZ DEFAULT NOW()

INDEX(task_id)
```

---

### Bảng `annotation_history`  ← audit trail
```sql
id             UUID PRIMARY KEY DEFAULT gen_random_uuid()
annotation_id  UUID REFERENCES annotations(id) ON DELETE CASCADE
changed_by     UUID REFERENCES users(id)
previous_data  JSONB NOT NULL          -- snapshot annotation_data trước khi sửa
change_reason  VARCHAR(50)             -- 'reviewer_edit' | 'labeler_revision' | 'undo'
created_at     TIMESTAMPTZ DEFAULT NOW()

INDEX(annotation_id)
```
> Insert 1 row vào đây (application-level) trước mỗi `UPDATE annotations`.
> Đọc lịch sử: `SELECT * FROM annotation_history WHERE annotation_id = ? ORDER BY created_at DESC`.

---

### Bảng `export_jobs`
```sql
id             UUID PRIMARY KEY DEFAULT gen_random_uuid()
project_id     UUID REFERENCES projects(id) ON DELETE CASCADE
requested_by   UUID REFERENCES users(id)
format         VARCHAR(10) NOT NULL   -- 'coco' | 'yolo' | 'csv' | 'json'
status         VARCHAR(20) DEFAULT 'pending'
               -- 'pending' | 'processing' | 'completed' | 'failed'
filters        JSONB DEFAULT '{}'
               -- {"status": "approved", "label_ids": [...], "date_from": "..."}
file_path      VARCHAR(500)           -- path download khi hoàn thành
error_message  TEXT
created_at     TIMESTAMPTZ DEFAULT NOW()
completed_at   TIMESTAMPTZ
```

---

**Lý do dùng JSONB cho `annotation_data`**: 4 loại annotation có cấu trúc khác nhau hoàn toàn. Dùng 1 cột JSONB thay vì 4 bảng riêng giúp truy vấn đơn giản hơn và dễ thêm loại annotation mới (keypoints, 3D box...) mà không cần migration schema.

---

## API Endpoints chính

```
POST /auth/login, /auth/register, /auth/refresh
GET  /auth/me

GET  /files/token                        ← cấp short-lived access token cho 1 file path
GET  /files/{access_token}               ← serve file (local) hoặc redirect pre-signed URL (S3)

GET/POST/PATCH/DELETE /projects
GET  /projects/{id}/stats

GET/POST/PATCH/DELETE /projects/{id}/labels
GET/POST  /projects/{id}/items
POST      /projects/{id}/items/upload         ← multipart images
POST      /projects/{id}/items/upload-text
GET       /projects/{id}/items/next-unlabeled

GET/POST/PATCH/DELETE /projects/{id}/items/{iid}/annotations
GET       /annotations/{id}/history                           ← audit trail theo version

GET/POST  /projects/{id}/tasks
POST      /projects/{id}/tasks/assign
POST      /projects/{id}/tasks/auto-assign
GET       /projects/{id}/tasks/my-queue

GET       /projects/{id}/reviews/queue
POST      /projects/{id}/reviews

POST      /projects/{id}/export
GET       /projects/{id}/export/{job_id}
GET       /projects/{id}/export/{job_id}/download
```

---

## Frontend - Các trang chính

| Route | Trang |
|---|---|
| `/dashboard` | Grid project cards (tên, loại, % hoàn thành) |
| `/projects/new` | Form tạo project (multi-step: info → labels → members) |
| `/projects/[id]` | Tab bar: Overview, Data, Labels, Tasks, Review, Export, Settings |
| `/projects/[id]/label/[itemId]` | Labeling workspace (full-screen canvas + panel) |
| `/projects/[id]/review` | Review queue (cùng workspace, read-only + approve/reject) |
| `/admin/users` | Quản lý user, role, invite |

---

## Libraries quan trọng

**Frontend:**
- `konva` + `react-konva` — canvas cho bounding box, polygon (có Transformer built-in)
- `zustand` — annotation session state + undo stack
- `@dnd-kit/sortable` — drag reorder label classes
- `react-dropzone` — upload file
- `react-hook-form` + `zod` — form validation
- `@tanstack/react-table` — table phân trang
- `recharts` — progress charts
- `shadcn/ui` — UI components base

**Backend:**
- `fastapi`, `uvicorn`, `sqlalchemy[asyncio]`, `asyncpg`, `alembic`
- `python-jose[cryptography]`, `passlib[bcrypt]` — JWT auth
- `Pillow` — thumbnail generation, image validation
- `python-magic` — server-side MIME validation
- `pycocotools` — COCO export format

---

## Kế hoạch theo Phase

### Phase 1: Core Foundation (Tuần 1–3)
- Setup FastAPI + SQLAlchemy + Alembic migrations
- Models: users, projects, project_members, label_classes, data_items
  - **[Fix]** Initial migration phải bao gồm `GIN INDEX` trên `data_items.metadata`
- JWT auth (register, login, refresh, /me)
- CRUD projects + label classes
- Upload endpoint (images + text), thumbnail generation
- Browse items endpoint (paginated, filter by status)
- **[Fix]** `storage_service.py` thiết kế ngay interface `generate_access_token(file_path, ttl)`:
  - Local: trả JWT ngắn hạn chứa `path` claim, serve qua `GET /files/{token}`
  - S3 (sau này): trả pre-signed URL thật — frontend không thay đổi gì
- Frontend: Login/Register, Dashboard, Project creation, Label schema editor, Upload zone + item grid
  - Mọi `<img src>` phải dùng `GET /files/token` → không hardcode file path

**Deliverable**: Tạo project, định nghĩa labels, upload data, browse items.

### Phase 2: Labeling Interfaces (Tuần 4–7)
- Backend: annotations table, POST/GET/PATCH/DELETE annotations, validation Pydantic theo loại
  - **[Fix]** Migration annotations phải bao gồm cột `version INTEGER DEFAULT 1` và `GIN INDEX` trên `annotation_data`
  - **[Fix]** `PATCH /annotations/{id}` nhận thêm field `version` từ client; backend trả `409 Conflict` nếu version lệch
- Frontend Labeling Workspace:
  - **Image Classification**: ClassificationPanel, keyboard shortcuts (1/2/3...)
  - **Object Detection**: BoundingBoxTool trên Konva, Transformer để resize
  - **[Fix] Segmentation**: PolygonTool + filled overlay — tách `ImageCanvas` thành **3 layer cố định**:
    - Layer 1 (bottom): `KonvaImage` — ảnh tĩnh, không bao giờ redraw
    - Layer 2 (middle): các annotation đã commit — chỉ redraw khi danh sách thay đổi
    - Layer 3 (top): tool đang tương tác (polygon/box đang vẽ) — redraw liên tục theo mouse
  - **Text NLP**: TextLabelPanel với span highlight
- NavigationBar (prev/next/skip/submit), LabelList sidebar
- Zustand store với undo stack
- Auto-save khi navigate

**Deliverable**: Label được cả 4 loại dữ liệu.

### Phase 3: Multi-User & Team (Tuần 8–10)
- Backend: task assignment (round-robin auto-assign), RBAC middleware, stats endpoint
- Frontend: Members tab, Task management table, My Queue view, Progress dashboard
- **[Fix]** Bổ sung SSE endpoint `GET /projects/{id}/events` — push real-time khi task status thay đổi
  - Dashboard và Review Queue subscribe SSE thay vì polling API mỗi N giây

**Deliverable**: Manager assign việc, nhiều labeler làm song song.

### Phase 4: Export & QA/Review (Tuần 11–13)
- Backend: reviews table, review queue logic, export service (COCO/YOLO/CSV/JSON), async export jobs
  - **[Fix]** Migration tạo bảng `annotation_history` trước khi build review logic
  - **[Fix]** Đăng ký SQLAlchemy `before_update` Event Listener trên model `Annotation` — tự động ghi `annotation_history` mà không cần gọi thủ công trong service. `changed_by` và `reason` truyền qua `SessionLocal` execution options hoặc context var.
- Frontend: Review workspace, Approve/Reject UI, Export page với format cards + filter, download history
  - Review workspace hiển thị nút "Xem lịch sử" — gọi `GET /annotations/{id}/history` để so sánh các version

**Deliverable**: End-to-end pipeline: upload → assign → label → review → export.

---

## Files quan trọng nhất (bắt đầu từ đây)

1. `backend/app/models/annotation.py` — JSONB polymorphic design + `version` column (quyết định kiến trúc lớn nhất)
2. `backend/app/deps.py` — RBAC dependency injection (ảnh hưởng mọi endpoint)
3. `backend/app/services/storage_service.py` — abstraction layer local/S3 + `generate_access_token()`
4. `backend/app/routers/files.py` — serve file qua token (pre-signed URL pattern)
5. `frontend/src/store/annotationStore.ts` — Zustand store (undo, tool state, unsaved)
6. `frontend/src/components/labeling/ImageCanvas.tsx` — Konva wrapper: 3-layer architecture (phức tạp nhất UI)
7. `backend/app/services/export_service.py` — COCO/YOLO/CSV/JSON exporters
8. `backend/app/services/history_service.py` — logic ghi `annotation_history`, được trigger tự động qua SQLAlchemy `before_update` Event Listener trên model `Annotation`

---

## Verification
1. Tạo project → định nghĩa 3 labels → upload 10 ảnh → label từng ảnh → export COCO JSON
2. Thêm 2 user → assign task → login labeler → label → submit → login reviewer → approve
3. Test NLP: upload text items → label spans → export CSV
4. Test auto-assign cho 100 items với 3 labelers
