# Project Rules — Data Labeling Web App

## 1. Nguyên tắc chung

- Không thêm abstraction, feature, hoặc error handling cho những trường hợp chưa xảy ra.
- Không viết comment giải thích WHAT code làm — chỉ comment WHY khi logic không hiển nhiên.
- Không tạo file mới khi có thể edit file hiện tại.
- Tuân thủ Phase roadmap trong `plan.md` — không implement Phase N+1 khi Phase N chưa xong.

---

## 2. Backend (FastAPI + SQLAlchemy async)

### Cấu trúc bắt buộc
- Mỗi domain có đúng 1 file trong mỗi layer: `models/`, `schemas/`, `routers/`, `services/`.
- Router chỉ gọi service — không viết business logic trực tiếp trong router.
- Service không import trực tiếp `Request`/`Response` của FastAPI — giữ service thuần Python.

### Database
- **Mọi migration Alembic phải chạy được cả `upgrade` lẫn `downgrade`.**
- `annotation_data` (bảng `annotations`) và `metadata` (bảng `data_items`) là JSONB — **luôn có GIN index** kể từ migration đầu tiên tạo bảng đó.
- Audit trail tự động qua **SQLAlchemy Event Listener** — không bắt dev tự gọi thủ công:
  ```python
  @event.listens_for(Annotation, "before_update")
  def _auto_save_history(mapper, connection, target):
      history_service.save_history_sync(connection, target)
  ```
  Không gọi `_save_history()` rải rác trong service — ORM tự trigger, không bỏ sót.
- Optimistic locking: mọi endpoint `PATCH /annotations/{id}` phải kiểm tra `version` từ client, trả `409 Conflict` nếu lệch.
- **Transaction scope**: Mọi thao tác ghi liên quan đến nhiều bảng (VD: duyệt Task + update DataItem + sinh ExportJob) phải wrap trong `async with session.begin()`. Không commit rải rác trong service.
- Dùng `TIMESTAMPTZ` (có timezone) cho tất cả cột thời gian — không dùng `TIMESTAMP`.
- Soft delete dùng `deleted_at TIMESTAMPTZ` — không `DELETE` trực tiếp row `annotations` và `data_items`.

### Async
- Mọi hàm trong `services/` và `routers/` phải `async def`.
- Không dùng `session.execute(...).fetchall()` — dùng `(await session.execute(...)).scalars().all()`.

### Auth & RBAC
- RBAC được inject qua `deps.py` — không kiểm tra role thủ công trong service.
- Không lưu thông tin nhạy cảm vào JWT payload ngoài `user_id`, `role`, `exp`.
- Token refresh dùng httpOnly cookie — access token dùng Authorization header.

### File Serving (bắt buộc)
- **Tuyệt đối không expose `file_path` ra response API.**
- Mọi endpoint trả về URL file phải dùng `storage_service.generate_access_token(file_path, ttl=3600)`.
- Frontend luôn gọi `GET /files/token?path=...` để lấy token, sau đó dùng `GET /files/{token}`.
- Local storage: token là JWT ngắn hạn chứa `path` claim.
- S3 (tương lai): `generate_access_token()` trả pre-signed URL thật — frontend không thay đổi gì.

---

## 3. Database Schema — những điều không được thay đổi

- `annotation_data` là JSONB polymorphic duy nhất — không tạo thêm bảng riêng cho từng loại annotation.
- `version INTEGER DEFAULT 1` trên bảng `annotations` — không xóa, không rename.
- Bảng `annotation_history` phải tồn tại khi review workflow hoạt động.
- `label_classes.attributes` là JSONB array — không normalize thành bảng riêng.

---

## 4. Frontend (Next.js 14 App Router + TypeScript strict)

### TypeScript
- `strict: true` trong `tsconfig.json` — không dùng `any`, không dùng `// @ts-ignore`.
- Mọi response từ API phải có Zod schema validation trước khi dùng.

### Konva Canvas — ImageCanvas.tsx (quan trọng nhất)
- **Bắt buộc tách thành đúng 3 layer:**
  1. `Layer` bottom: `KonvaImage` — ảnh tĩnh, `listening={false}`, không redraw khi user tương tác.
  2. `Layer` middle: các annotation đã commit — chỉ re-render khi danh sách annotation thay đổi.
  3. `Layer` top: tool đang active (polygon/box đang vẽ) — redraw tự do theo mouse event.
- Không đặt tất cả shape vào 1 layer duy nhất — sẽ drop FPS với 50+ annotation.
- **Shape đã commit ở Layer middle**: bắt buộc set `perfectDrawEnabled={false}` và `listening={false}` mặc định. Chỉ bật `listening={true}` khi user hover/click để edit. Tăng đáng kể FPS khi canvas có 500+ bounding box.
- Polygon với > 500 điểm: chuyển tính toán snap/intersect vào Web Worker.

### Zustand Store
- `annotationStore` chứa: danh sách annotation hiện tại, undo stack, active tool, unsaved flag.
- Không fetch API trực tiếp trong store — store chỉ giữ state, component gọi API rồi dispatch vào store.
- Undo stack tối đa 50 bước — không giới hạn vô tận.

### File/Image Display
- Không dùng `<img src="/uploads/...">` hay bất kỳ hardcoded path nào.
- Luôn dùng hook `useFileUrl(filePath)` — hook này gọi `/files/token` và cache URL.

### State & Data Fetching
- Dùng `@tanstack/react-query` cho server state — không `useState` + `useEffect` để fetch.
- Form dùng `react-hook-form` + `zod` — không validate thủ công.

---

## 5. API Contract

- Mọi response lỗi phải có shape: `{ "detail": "message", "code": "ERROR_CODE" }`.
- Pagination response: `{ "items": [...], "total": N, "page": P, "page_size": S }`.
- `PATCH` là partial update (chỉ gửi field cần thay đổi) — không dùng `PUT` cho annotation.
- `PATCH /annotations/{id}` body phải có `version: number` — backend trả `409` nếu stale.

### Xử lý 409 Conflict ở Frontend (bắt buộc)
Khi nhận HTTP `409` từ `PATCH /annotations/{id}`, frontend **phải** theo thứ tự:
1. Fetch lại annotation mới nhất từ server (`GET /annotations/{id}`).
2. Cập nhật Zustand store với dữ liệu mới (bao gồm `version` mới).
3. Hiển thị Toast thông báo chặn tương tác: *"Dữ liệu đã bị thay đổi bởi người khác, vui lòng review lại trước khi lưu."*

Không tự động merge hay overwrite — người dùng phải xem lại thay đổi trước.

---

## 6. Export Service

- Export COCO/YOLO/CSV/JSON chạy async qua `export_jobs` table — không block request.
- Export chỉ lấy annotation có `deleted_at IS NULL` và `tasks.status = 'approved'` (theo filter).
- File export lưu vào `storage/exports/` — không lưu vào cùng thư mục với data_items.

---

## 7. Real-time (Phase 3+)

- SSE endpoint `GET /projects/{id}/events` — dùng `StreamingResponse` của FastAPI.
- Không dùng WebSocket cho tính năng này — SSE đủ cho unidirectional status updates.
- Frontend subscribe SSE thay vì polling — không `setInterval(() => fetch(...), N)` cho task status.
- **HTTP/2 bắt buộc ở production**: HTTP/1.1 giới hạn tối đa 6 kết nối SSE/domain trên mỗi trình duyệt — user mở tab thứ 7 sẽ bị block. Reverse proxy (Caddy/Nginx) phải bật HTTP/2 (`h2`) để xóa giới hạn này. Ghi rõ trong `docker-compose.yml` production config.

---

## 8. Testing

- Verification cases trong `plan.md` là test tối thiểu phải pass trước khi kết thúc mỗi Phase.
- Không mock database trong integration test — dùng PostgreSQL test container.
- Unit test chỉ cho pure logic (export format converter, annotation validator) — không test ORM.

---

## 9. Git & Commit

- Commit message: `<type>(<scope>): <mô tả ngắn>` — type: `feat`, `fix`, `refactor`, `test`, `chore`.
- Mỗi commit là 1 unit của công việc — không commit cả Phase trong 1 lần.
- Không commit file `.env`, secret, hoặc file upload thật vào repo.
- Migration file được commit cùng model thay đổi — không tách thành commit riêng.
