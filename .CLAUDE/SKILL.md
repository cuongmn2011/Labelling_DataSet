# Project Skills — Data Labeling Web App

Tập hợp các quy trình lặp đi lặp lại trong dự án. Khi được yêu cầu thực hiện một trong các tác vụ dưới đây, hãy làm theo đúng thứ tự các bước.

---

## SKILL: new-migration

**Trigger:** "tạo migration", "thêm cột X", "thêm bảng Y", "sửa schema"

**Bước thực hiện:**

1. Cập nhật SQLAlchemy model trong `backend/app/models/` trước.
2. Tạo migration tự động:
   ```bash
   cd backend && alembic revision --autogenerate -m "<mô tả ngắn>"
   ```
3. Mở file migration vừa tạo trong `alembic/versions/` — kiểm tra:
   - `upgrade()` đúng với thay đổi model chưa.
   - `downgrade()` có thể rollback hoàn toàn không.
   - Nếu có cột JSONB mới → thêm GIN index thủ công (autogenerate không tự tạo GIN index):
     ```python
     op.execute("CREATE INDEX idx_<table>_<col> ON <table> USING GIN (<col>)")
     # downgrade: op.execute("DROP INDEX idx_<table>_<col>")
     ```
4. Chạy kiểm tra:
   ```bash
   alembic upgrade head && alembic downgrade -1 && alembic upgrade head
   ```
5. Commit migration file cùng model trong 1 commit — không tách riêng.

---

## SKILL: new-router

**Trigger:** "thêm endpoint", "tạo router mới", "API cho X"

**Bước thực hiện:**

1. Tạo `backend/app/routers/<domain>.py` theo cấu trúc:
   ```python
   router = APIRouter(prefix="/<domain>", tags=["<domain>"])

   @router.get("/")
   async def list_items(
       db: AsyncSession = Depends(get_db),
       current_user: User = Depends(require_role("labeler")),
   ):
       return await <domain>_service.list(db, ...)
   ```
2. Tạo `backend/app/services/<domain>_service.py` — logic nghiệp vụ ở đây, không trong router.
3. Tạo `backend/app/schemas/<domain>.py` — Pydantic request/response schemas.
4. Đăng ký router trong `backend/app/main.py`:
   ```python
   app.include_router(<domain>.router)
   ```
5. Kiểm tra response shape: lỗi phải có `{"detail": "...", "code": "..."}`, list phải có pagination wrapper.

---

## SKILL: new-annotation-type

**Trigger:** "thêm loại annotation mới", "hỗ trợ keypoint", "hỗ trợ 3D box"

**Bước thực hiện:**

1. **Backend — Schema Pydantic** (`backend/app/schemas/annotation.py`):
   - Tạo class mới kế thừa `BaseAnnotationData`, ví dụ `KeypointAnnotationData`.
   - Thêm vào `AnnotationDataUnion = Union[..., KeypointAnnotationData]`.

2. **Backend — Validator** (`backend/app/services/annotation_service.py`):
   - Thêm case cho loại mới vào hàm `validate_annotation_data(project_type, data)`.

3. **DB** — Không cần migration schema. JSONB `annotation_data` tự xử lý cấu trúc mới.
   - Chỉ cập nhật comment trong `plan.md` phần bảng `annotations` để document shape mới.

4. **Frontend — Canvas Tool** (`frontend/src/components/labeling/`):
   - Tạo `<NewTypeTool>.tsx` mới — không sửa `ImageCanvas.tsx`, chỉ import tool vào đó.
   - Tool hoạt động trên **Layer 3** (top layer) trong `ImageCanvas.tsx`.
   - Khi commit annotation → chuyển shape sang Layer 2.

5. **Frontend — Store** (`frontend/src/store/annotationStore.ts`):
   - Thêm action `addAnnotation` xử lý type mới — undo stack tự động nhờ kiến trúc hiện có.

6. **Export Service** (`backend/app/services/export_service.py`):
   - Thêm handler cho loại mới trong mỗi format cần thiết (COCO, YOLO...).

---

## SKILL: new-export-format

**Trigger:** "thêm format export", "export Pascal VOC", "export TFRecord"

**Bước thực hiện:**

1. Thêm value mới vào `format` enum trong schema `export_jobs` — tạo migration nếu dùng `CHECK` constraint.

2. Tạo class exporter trong `backend/app/services/export_service.py`:
   ```python
   class PascalVOCExporter(BaseExporter):
       def export(self, annotations: list[Annotation]) -> bytes: ...
   ```
   Kế thừa `BaseExporter` — không viết logic lặp lại (query DB, tạo file zip...).

3. Đăng ký vào `EXPORTER_REGISTRY`:
   ```python
   EXPORTER_REGISTRY = {
       "coco": COCOExporter,
       "yolo": YOLOExporter,
       "pascal_voc": PascalVOCExporter,  # thêm ở đây
   }
   ```

4. Frontend — `Export page`: thêm card format mới với tên/mô tả, không cần thay đổi logic khác.

---

## SKILL: add-label-attribute

**Trigger:** "thêm thuộc tính cho label", "label có thêm field X"

**Bước thực hiện:**

1. Không cần migration — `label_classes.attributes` là JSONB array.
2. Cập nhật Pydantic schema `LabelAttributeSchema` nếu thêm kiểu attribute mới (`type: "range"`, `type: "color"`...).
3. Frontend — Label Schema Editor: thêm UI control tương ứng cho kiểu mới.
4. Frontend — Labeling Workspace: `LabelList` sidebar render control theo `attribute.type` — thêm case mới vào switch.

---

## SKILL: dev-start

**Trigger:** "chạy dev", "khởi động project", "start local"

**Bước thực hiện:**

```bash
# Lần đầu hoặc sau khi thay đổi deps
docker compose build

# Khởi động toàn bộ stack
docker compose up

# Hoặc chạy background
docker compose up -d && docker compose logs -f
```

Hot-reload hoạt động tự động:
- Backend: `./backend/app/` được mount vào container, uvicorn `--reload` theo dõi thay đổi.
- Frontend: `./frontend/src/` và `./frontend/public/` được mount, `WATCHPACK_POLLING=true` đảm bảo Next.js phát hiện thay đổi trên macOS.
- Migration: `alembic upgrade head` chạy tự động khi backend container khởi động.

Kiểm tra sau khi start:
- `GET http://localhost:8000/health` → `{"status": "ok"}`
- `GET http://localhost:3000` → Login page render

Lệnh thường dùng khi đang phát triển:
```bash
# Xem log từng service
docker compose logs -f backend
docker compose logs -f frontend

# Chạy lệnh trong container (ví dụ: tạo migration)
docker compose exec backend alembic revision --autogenerate -m "add users table"

# Seed dữ liệu
docker compose exec backend python -m app.utils.seed

# Rebuild 1 service sau khi thay đổi Dockerfile hoặc deps
docker compose up -d --build backend
```

---

## SKILL: seed-db

**Trigger:** "seed data", "tạo dữ liệu test", "tạo user demo"

**Bước thực hiện:**

```bash
cd backend
python -m app.utils.seed  # file này cần tạo nếu chưa có
```

Seed tối thiểu phải tạo:
- 1 admin user: `admin@demo.com / admin123`
- 1 labeler: `labeler@demo.com / labeler123`
- 1 reviewer: `reviewer@demo.com / reviewer123`
- 1 project loại `object_detection` với 3 label classes
- 10 data items (ảnh placeholder hoặc text)

---

## SKILL: verify-phase

**Trigger:** "kiểm tra phase X xong chưa", "verify phase"

**Bước thực hiện:** Chạy từng test case tương ứng trong `plan.md` mục **Verification**.

| Phase | Test case cần pass |
|---|---|
| Phase 1 | Tạo project → định nghĩa 3 labels → upload 10 ảnh → browse items grid |
| Phase 2 | Label đủ 4 loại → undo/redo hoạt động → auto-save khi navigate |
| Phase 3 | Thêm 2 user → assign task → login labeler → submit → login reviewer → approve |
| Phase 4 | Export COCO JSON → export YOLO → test NLP span export CSV |

Không kết thúc phase nếu bất kỳ test case nào fail.

---

## SKILL: check-security

**Trigger:** "kiểm tra bảo mật", "security check"

Kiểm tra nhanh các điểm dễ quên:

- [ ] Không có endpoint nào trả `file_path` raw trong response.
- [ ] Mọi `<img>` trong frontend dùng URL từ `/files/{token}`, không phải path tĩnh.
- [ ] JWT access token TTL ≤ 15 phút, refresh token trong httpOnly cookie.
- [ ] Upload endpoint validate MIME type server-side bằng `python-magic` (không tin vào Content-Type header).
- [ ] Mọi endpoint cần auth đều có `Depends(get_current_user)` — không có endpoint "open by accident".
- [ ] CORS chỉ cho phép origin của frontend — không `allow_origins=["*"]` trên production.
