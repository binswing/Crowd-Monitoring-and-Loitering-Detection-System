# Pipeline Giám Sát Đám Đông và Phát Hiện Bất Thường (Crowd Tracking & Loitering Detection)

## Tổng Quan Dự Án
Dự án này triển khai hệ thống giám sát thời gian thực sử dụng trí tuệ nhân tạo để phát hiện, theo dõi, đếm người đi qua khu vực địa lý (Geofencing) và phát hiện hành vi lảng vảng (Loitering). Hệ thống kết hợp sức mạnh của YOLOv8 (phát hiện đối tượng) và thuật toán ByteTrack (theo dõi đa đối tượng) với logic kiểm tra dựa trên vận tốc và khu vực để phát hiện bất thường.

### Các Tính Năng Được Cải Tiến & Mô-đun Hóa
- **Theo dõi đối tượng (Tracking):** Sử dụng YOLOv8 và ByteTrack để phát hiện và gán ID ổn định cho mỗi người.
- **Geofencing (Hàng rào ảo):** Đếm số lượng người thuộc khu vực địa lý (Polygon) linh hoạt.
- **Phát hiện lảng vảng (Loitering):** Phát hiện người đứng quá lâu hoặc bám trụ tại một khu vực với tốc độ di chuyển cực thấp dựa vào thời gian định mức.
- **Thiết kế mô-đun:** Dễ dàng bảo trì, phát triển và chạy riêng biệt từng chức năng qua cấu trúc lệnh trên Terminal.

## Yêu Cầu Hệ Thống
- Python 3.8+
- Webcam hoặc camera IP (cho video thời gian thực)
- RAM: 4GB+ (khuyến nghị 8GB)
- GPU: Tùy chọn (cho tốc độ cao hơn)

## Cài Đặt

### 1. Clone Repository
```bash
https://github.com/binswing/Crowd-Monitoring-and-Loitering-Detection-System.git .
```

### 2. Tạo Virtual Environment
```bash
python3 -m venv venv
# Trên Windows:
venv\Scripts\activate
# Trên Linux/Mac:
source venv/bin/activate
```

### 3. Cài Đặt Dependencies
```bash
pip install -r requirements.txt
```

## Hướng Dẫn Sử Dụng

Mã nguồn chính của hệ thống nằm trong thư mục `BytetrackCountingLoitering/` và phần web backend nằm trong thư mục `web/`.

Để khởi động hệ thống (Redis, backend và các Celery worker), chạy các lệnh sau trên 4 terminal riêng biệt:

Run:

```bash
# Terminal 1 — Redis:
docker compose -f web\docker-compose.yml up -d redis

# Terminal 2 — Backend:
python -m uvicorn web.app.main:app --host 0.0.0.0 --port 8000 --reload

# Terminal 3 — Celery (stream_queue):
python -m celery -A web.app.core.celery_app worker --loglevel=INFO --pool=solo -Q stream_queue

# Terminal 4 — Celery (alert/notification/hardware/stats queues):
python -m celery -A web.app.core.celery_app worker --loglevel=INFO --pool=solo -Q alert_queue,notification_queue,hardware_queue,stats_queue
```

Ghi chú:
- Chạy Redis trước khi start backend và worker.
- Trên Windows, giữ các terminal mở để xem log; dùng PowerShell hoặc CMD được cấu hình môi trường ảo nếu cần.

## Cấu Trúc Dự Án
Tóm tắt cấu trúc thư mục chính trong repository (chỉ liệt kê các mục quan trọng):

```
.
├── BytetrackCountingLoitering/    # Pipeline tracking & loitering (core)
├── CrowdCounting/                  # Các mô-đun đếm thử nghiệm
├── web/                            # Web backend, API, Celery tasks, frontend assets
│   ├── app/
│   ├── frontend/
│   └── docker-compose.yml
├── requirements.txt
├── README.md
└── yolov8s.pt                       # Mô hình YOLOv8 mẫu
```

Thư mục `web/` chứa backend FastAPI, Celery worker, cấu hình Docker và frontend tĩnh (`web/frontend/`).

Chi tiết cấu trúc bên trong `web/app/`:

```
web/app/
├── __init__.py
├── main.py                  # FastAPI app entrypoint
├── adapters/                # Adapter cho AI, camera, hardware, notifier
│   ├── ai/                  # Các implementation AI (factory, mock, model wrappers)
│   ├── camera/              # Camera adapters (video file, mock, hardware clients)
│   ├── hardware/            # Giao tiếp với thiết bị phần cứng
│   └── notifier/            # Notifier implementations (e.g., Telegram)
├── api/                     # Các route FastAPI (routes_*.py)
├── core/                    # Core app: `celery_app.py`, `config.py`, `database.py`, `redis.py`
├── models/                  # ORM/DB models (e.g., SQLAlchemy)
├── schemas/                 # Pydantic schemas cho request/response
├── services/                # Business logic và service layer
├── tasks/                   # Celery tasks (alert_tasks, stream_tasks, stats_tasks,...)
├── ws/                      # WebSocket routes/handlers
└── static/                  # Static assets: `app.js`, `style.css`, snapshots/
```

Gợi ý chỉnh sửa/quan sát nhanh:
- Entrypoint API: `web/app/main.py` (khởi tạo FastAPI + mount các router).
- Cấu hình Celery: `web/app/core/celery_app.py` và các task trong `web/app/tasks/`.
- Thêm/điều chỉnh biến môi trường trong `web/.env` và `web/.env.example`.
- Nếu muốn mở rộng adapter hoặc thêm camera mới, bắt đầu từ `web/app/adapters/camera/`.

## Ghi Chú Quan Trọng
- **Docker & Redis:** file `web/docker-compose.yml` có dịch vụ `redis` — khởi động Redis trước khi chạy backend/celery (ví dụ lệnh trong phần Hướng Dẫn Sử Dụng).
- **Biến môi trường:** sao chép `web/.env.example` sang `web/.env` và điều chỉnh nếu cần trước khi khởi động services.
- **Chạy trên Windows:** dùng PowerShell/ CMD đã kích hoạt virtualenv hoặc Docker Desktop; giữ các terminal mở để xem logs.
- **Celery:** Celery dùng Redis làm broker (mặc định) — đảm bảo Redis reachable từ worker; các queue được tách theo nhiệm vụ (stream_queue, alert_queue, ...).
- **Tuỳ chỉnh tham số:** `BytetrackCountingLoitering/config.py` chứa các ngưỡng, tọa độ polygon và tham số loitering.

## Kế Hoạch Phát Triển (To-do)
- **1. Dockerize full stack (high):** tạo image cho backend, worker và hướng dẫn deploy (compose/stack).
- **2. API/Streaming (high):** bổ sung API trả kết quả stream theo thời gian thực và websocket cho frontend.
- **3. Tự động hóa & CI (medium):** tests unit cho pipeline, linting, pipeline CI/CD cơ bản.
- **4. Quản lý mô hình (medium):** thêm script tải/kiểm tra phiên bản mô hình, storage cho checkpoints.
- **5. Cải thiện phát hiện (low):** thêm logic phát hiện ngã, chạy tán loạn, tối ưu tham số theo dữ liệu thực tế.

## Giấy Phép
Dự án sử dụng đa phần là các thư viện mã nguồn mở phân phối miễn phí theo quy định của YOLOv8 và chuẩn thư viện Python mở khác. Vui lòng tuân thủ bản quyền của thư viện bên thứ ba đang được áp dụng.
