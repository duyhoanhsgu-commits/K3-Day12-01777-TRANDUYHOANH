# Giải Thích Chi Tiết 4 Checkpoint (CP1 – CP4) — K3 Ngày 12

Tài liệu này tổng hợp và giải thích toàn bộ kiến trúc, lý thuyết thiết kế cùng toàn bộ mã nguồn thực tế đã được triển khai cho 4 Checkpoint từ **CP1** đến **CP4** trong hệ thống **Day 12 Production AI Agent**.

---

## MỤC LỤC
1. [Checkpoint 1 (CP1): 12-Factor Config, Health Check & Structured Logging](#checkpoint-1-cp1-12-factor-config-health-check--structured-logging)
2. [Checkpoint 2 (CP2): Dockerization & Container Security](#checkpoint-2-cp2-dockerization--container-security)
3. [Checkpoint 3 (CP3): API Security — Auth, Rate Limiter & Cost Guard](#checkpoint-3-cp3-api-security--auth-rate-limiter--cost-guard)
4. [Checkpoint 4 (CP4): Scaling & Reliability — Stateless, Readiness & Graceful Shutdown](#checkpoint-4-cp4-scaling--reliability--stateless-readiness--graceful-shutdown)
5. [Bảng Tổng Hợp Kiểm Thử & Chấm Điểm](#bảng-tổng-hợp-kiểm-thử--chấm-điểm)

---

## Checkpoint 1 (CP1): 12-Factor Config, Health Check & Structured Logging

### 1.1 Mục tiêu kiến trúc
- **12-Factor App (Yếu tố Config)**: Code phải độc lập và không đổi trên mọi môi trường (Laptop, Staging, Production). Cấu hình và bí mật (secrets) được nạp hoàn toàn qua biến môi trường.
- **Fail-Fast trên Secrets**: Biến nhạy cảm như `AGENT_API_KEY` **không được có giá trị mặc định**. Nếu người vận hành quên cấu hình key trên cloud, container phải dừng ngay lập tức khi khởi động (crash lúc deploy) thay vì âm thầm chạy với key giả tạo lỗ hổng bảo mật.
- **Structured Logging**: Thay thế `print()` thông thường bằng định dạng JSON trên đúng một dòng duy nhất (`ndjson`). Điều này giúp các hệ thống giám sát tập trung (Datadog, Grafana, CloudWatch, Loki) dễ dàng parse, filter và dựng dashboard tự động.
- **Liveness Probe (`/health`)**: Cung cấp endpoint siêu nhẹ để orchestrator kiểm tra xem tiến trình Python/Uvicorn còn sống hay không. Tuyệt đối **không kiểm tra Redis/DB** tại endpoint này để tránh tình trạng Redis nghẽn làm cả cụm container bị restart oan.

---

### 1.2 Mã nguồn triển khai

#### File `app/config.py`
```python
"""CP1 — Cấu hình theo 12-Factor."""

from __future__ import annotations

from functools import lru_cache

from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    """Toàn bộ cấu hình của service đọc từ biến môi trường."""

    port: int = 8000
    agent_api_key: str  # Bắt buộc — không có mặc định (Fail-fast)
    redis_url: str = "redis://localhost:6379/0"
    rate_limit_per_minute: int = 10
    monthly_budget_usd: float = 10.0
    log_level: str = "INFO"

    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        extra="ignore",
    )


@lru_cache(maxsize=1)
def get_settings() -> Settings:
    """Đọc cấu hình một lần rồi cache lại."""
    return Settings()
```

#### File `app/logging_utils.py`
```python
"""CP1 — Structured logging."""

from __future__ import annotations

import json
from datetime import datetime, timezone


def utc_now_iso() -> str:
    """Thời điểm hiện tại theo ISO-8601, múi giờ UTC."""
    return datetime.now(timezone.utc).isoformat()


def log_event(event: str, level: str = "info", **fields) -> str:
    """Ghi một dòng log JSON ra stdout."""
    payload = {
        "event": event,
        "level": level.lower(),
        "timestamp": utc_now_iso(),
        **fields,
    }
    line = json.dumps(payload, ensure_ascii=False)
    print(line, flush=True)
    return line
```

#### File `app/main.py` (Phần Liveness Probe)
```python
@app.get("/health")
def health():
    """Liveness probe — process còn sống không?"""
    if lifecycle.shutting_down:
        return JSONResponse(status_code=503, content={"status": "shutting_down"})
    return {"status": "ok", "service": SERVICE_NAME, "version": SERVICE_VERSION}
```

---

## Checkpoint 2 (CP2): Dockerization & Container Security

### 2.1 Mục tiêu kiến trúc
- **Multi-stage Build**: Chia thành 2 stage:
  1. `builder`: Cài đặt thư viện phụ thuộc từ `requirements.txt` vào thư mục tạm `/install`.
  2. `runtime` (`python:3.11-slim`): Chỉ copy kết quả đã biên dịch sang, không mang theo trình biên dịch (compiler) hay công cụ build dư thừa, giảm dung lượng image xuống còn ~64MB (nén) / ~270MB (giải nén).
- **Docker Layer Caching**: Luôn `COPY requirements.txt` và chạy `pip install` trước khi `COPY app/`. Khi chỉ sửa mã nguồn, Docker sẽ sử dụng lại cache tầng thư viện giúp thời gian build chỉ mất vài giây.
- **Bảo mật Non-root**: Sử dụng lệnh `useradd` để tạo user `appuser` (UID 1001) và chuyển quyền thực thi với chỉ thị `USER appuser`. Ngăn chặn kẻ tấn công chiếm quyền root trên host nếu ứng dụng bị khai thác qua lỗ hổng RCE.
- **Tương thích cổng động trên Cloud**: Đọc cổng qua biến môi trường `$PORT` (ví dụ `CMD ["sh", "-c", "exec uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]`).
- **Bảo vệ Secret với `.dockerignore`**: Ngăn không cho file `.env`, `.git`, `.venv`, `__pycache__` bị đóng gói vào image đưa lên registry công khai.

---

### 2.2 Mã nguồn triển khai

#### File `Dockerfile`
```dockerfile
# ---- Stage 1: Builder ----
FROM python:3.11-slim AS builder
WORKDIR /build
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# ---- Stage 2: Production Runtime ----
FROM python:3.11-slim
WORKDIR /app
COPY --from=builder /install /usr/local
COPY app/ app/
COPY utils/ utils/

# Chạy dưới quyền user thường (bảo mật container)
RUN useradd -r -u 1001 appuser
USER appuser

EXPOSE 8000

# Health check dùng Python tích hợp sẵn (không cần curl)
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD python -c "import os, urllib.request; port = os.getenv('PORT', '8000'); urllib.request.urlopen(f'http://127.0.0.1:{port}/health', timeout=3)"

# Expand biến $PORT khi chạy trên cloud platform
CMD ["sh", "-c", "exec uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]
```

#### File `.dockerignore`
```dockerignore
.git
.gitignore
.env
.venv
__pycache__
*.pyc
*.pyo
*.pyd
.pytest_cache
tests
docs
```

#### File `docker-compose.yml`
```yaml
services:
  redis:
    image: redis:7-alpine
    command: ["redis-server", "--appendonly", "yes"]
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

  agent:
    build: .
    expose:
      - "8000"
    environment:
      AGENT_API_KEY: ${AGENT_API_KEY:?set AGENT_API_KEY in .env}
      REDIS_URL: redis://redis:6379/0
    depends_on:
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "python", "-c", "import os, urllib.request; port = os.getenv('PORT', '8000'); urllib.request.urlopen(f'http://127.0.0.1:{port}/health', timeout=3)"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 10s

  nginx:
    image: nginx:1.27-alpine
    ports:
      - "8000:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - agent

volumes:
  redis-data:
```

---

## Checkpoint 3 (CP3): API Security — Auth, Rate Limiter & Cost Guard

### 3.1 Mục tiêu kiến trúc
- **Xác thực an toàn (Timing Attack Defense)**: So sánh `X-API-Key` với secret cấu hình bằng `secrets.compare_digest()`. Toán tử so sánh thông thường (`==`) sẽ ngắt ngay ở ký tự sai đầu tiên, khiến thời gian phản hồi bị rò rỉ và có thể bị tấn công brute-force theo thời gian.
- **Rate Limiting — Thuật toán Cửa Sổ Trượt (Sliding Window)**:
  - Khác với Fixed Window (dễ bị bùng nổ 20 req tại giây 59 và 00), Sliding Window dùng Redis Sorted Set (ZSET) lưu mốc thời gian của từng request.
  - Mỗi request có member dạng `{now}:{uuid}` để đảm bảo tính duy nhất, không bị ghi đè khi 2 request đến trong cùng một mili-giây.
  - Quy trình xử lý theo thứ tự: **Prune** (xóa entry > 60s) ➔ **Count** (đếm bằng `zcard`) ➔ **Check** (nếu `>= limit` ném lỗi `429`, không ghi nhận request lỗi) ➔ **Record** (ghi request hợp lệ và đặt TTL).
- **Cost Guard — Quản lý Ngân Sách Token (Soft Quota)**:
  - Tích lũy chi phí theo tháng trong key `cost:{user_id}:{YYYY-MM}` qua lệnh `INCRBYFLOAT`.
  - Cơ chế **Pre-check + Post-record**: Kiểm tra `spent + estimated_cost > budget` trước khi gọi LLM (ném mã lỗi chuẩn `402 Payment Required`). Sau khi LLM trả về, ghi nhận chi phí thực tế vào Redis.

---

### 3.2 Mã nguồn triển khai

#### File `app/auth.py`
```python
"""CP3 — Xác thực bằng API key."""

from __future__ import annotations

import secrets

from fastapi import Header, HTTPException, status

from .config import get_settings

ANONYMOUS_USER = "anonymous"


def verify_api_key(
    x_api_key: str | None = Header(default=None),
    x_user_id: str | None = Header(default=None),
) -> str:
    """Kiểm tra header X-API-Key; trả về user_id nếu hợp lệ."""
    expected_key = get_settings().agent_api_key
    if not x_api_key or not secrets.compare_digest(x_api_key, expected_key):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="invalid or missing API key",
        )
    return x_user_id if x_user_id else ANONYMOUS_USER
```

#### File `app/rate_limiter.py`
```python
"""CP3 — Rate limiting bằng thuật toán sliding window."""

from __future__ import annotations

import time
import uuid

from fastapi import HTTPException, status

WINDOW_SECONDS = 60


class RateLimiter:
    def __init__(self, client, limit_per_minute: int) -> None:
        self.client = client
        self.limit = limit_per_minute

    @staticmethod
    def _key(user_id: str) -> str:
        """Mỗi user một key riêng."""
        return f"ratelimit:{user_id}"

    def hit_count(self, user_id: str, now: float | None = None) -> int:
        """Số request của user trong WINDOW_SECONDS giây gần nhất."""
        now = now if now is not None else time.time()
        key = self._key(user_id)
        self.client.zremrangebyscore(key, 0, now - WINDOW_SECONDS)
        return self.client.zcard(key)

    def check(self, user_id: str, now: float | None = None) -> None:
        """Cho qua nếu còn quota, ngược lại raise 429."""
        now = now if now is not None else time.time()
        if self.hit_count(user_id, now) >= self.limit:
            raise HTTPException(
                status_code=status.HTTP_429_TOO_MANY_REQUESTS,
                detail="rate limit exceeded",
                headers={"Retry-After": str(WINDOW_SECONDS)},
            )
        key = self._key(user_id)
        self.client.zadd(key, {f"{now}:{uuid.uuid4().hex}": now})
        self.client.expire(key, WINDOW_SECONDS)
```

#### File `app/cost_guard.py`
```python
"""CP3 — Cost guard: chặn chi phí trước khi hóa đơn chặn bạn."""

from __future__ import annotations

from datetime import datetime, timezone

from fastapi import HTTPException, status

KEY_TTL_SECONDS = 40 * 24 * 3600


class CostGuard:
    def __init__(self, client, monthly_budget_usd: float) -> None:
        self.client = client
        self.budget = monthly_budget_usd

    @staticmethod
    def current_month() -> str:
        """Nhãn tháng hiện tại dạng 'YYYY-MM' (UTC)."""
        return datetime.now(timezone.utc).strftime("%Y-%m")

    @classmethod
    def _key(cls, user_id: str, month: str | None = None) -> str:
        return f"cost:{user_id}:{month or cls.current_month()}"

    def spent(self, user_id: str, month: str | None = None) -> float:
        """Số tiền user đã tiêu trong tháng."""
        val = self.client.get(self._key(user_id, month))
        return float(val) if val is not None else 0.0

    def check(
        self,
        user_id: str,
        estimated_cost: float = 0.0,
        month: str | None = None,
    ) -> None:
        """Cho qua nếu còn ngân sách, ngược lại raise 402."""
        if self.spent(user_id, month) + estimated_cost > self.budget:
            raise HTTPException(
                status_code=status.HTTP_402_PAYMENT_REQUIRED,
                detail="monthly budget exceeded",
            )

    def record(self, user_id: str, cost: float, month: str | None = None) -> float:
        """Cộng dồn chi phí vừa phát sinh, trả về tổng mới."""
        key = self._key(user_id, month)
        total = self.client.incrbyfloat(key, cost)
        self.client.expire(key, KEY_TTL_SECONDS)
        return float(total)
```

---

## Checkpoint 4 (CP4): Scaling & Reliability — Stateless, Readiness & Graceful Shutdown

### 4.1 Mục tiêu kiến trúc
- **Stateless Backend**: Bộ nhớ hội thoại hoàn toàn nằm ở Redis List (`history:{user_id}`). Các container không lưu trạng thái trong RAM cục bộ, cho phép nhân bản ứng dụng lên N container (`--scale agent=N`) sau Nginx Load Balancer mà ngữ cảnh của user vẫn được duy trì liền mạch.
- **Memory Trimming & Expire**:
  - Dùng `ltrim(key, -20, -1)` để chỉ giữ tối đa 20 tin nhắn gần nhất, tránh prompt phình to làm đội chi phí token.
  - Tự động gán `expire(key, 7 ngày)` để Redis tự dọn dẹp bộ nhớ.
- **Readiness Probe (`/ready`)**: Kiểm tra xem container đã sẵn sàng phục vụ traffic hay chưa thông qua `store.ping()`. Nếu Redis gặp sự cố, endpoint trả về `503`, giúp Load Balancer tạm thời không định tuyến request vào container này (nhưng container không bị restart oan như `/health`).
- **Graceful Shutdown (Xử lý tín hiệu SIGTERM)**:
  - Khi triển khai phiên bản mới, orchestrator gửi `SIGTERM`. 
  - `Lifecycle` bắt tín hiệu, bật cờ `shutting_down = True` (khiến `/ready` và `/health` lập tức trả về `503` để Load Balancer ngắt traffic mới), sau đó ủy quyền lại cho handler của Uvicorn để xử lý nốt các request đang dang dở trước khi dừng tiến trình.

---

### 4.2 Mã nguồn triển khai

#### File `app/store.py`
```python
"""CP4 — Stateless: state sống ngoài process."""

from __future__ import annotations

import json
import redis
from .config import get_settings

HISTORY_MAX_MESSAGES = 20
HISTORY_TTL_SECONDS = 7 * 24 * 3600


def get_redis_client(url: str | None = None):
    url = url or get_settings().redis_url
    if url.startswith("fake://"):
        import fakeredis
        return fakeredis.FakeRedis(decode_responses=True)
    return redis.from_url(url, decode_responses=True)


class ConversationStore:
    """Lưu lịch sử hội thoại của từng user trong Redis List."""

    def __init__(self, client) -> None:
        self.client = client

    @staticmethod
    def _key(user_id: str) -> str:
        return f"history:{user_id}"

    def ping(self) -> bool:
        """Redis có trả lời không? Dùng cho endpoint /ready."""
        try:
            return bool(self.client.ping())
        except Exception:
            return False

    def append(self, user_id: str, role: str, content: str) -> None:
        """Ghi thêm một lượt vào lịch sử."""
        key = self._key(user_id)
        msg = json.dumps({"role": role, "content": content}, ensure_ascii=False)
        self.client.rpush(key, msg)
        self.client.ltrim(key, -HISTORY_MAX_MESSAGES, -1)
        self.client.expire(key, HISTORY_TTL_SECONDS)

    def get_history(self, user_id: str) -> list[dict]:
        """Đọc lịch sử hội thoại, cũ nhất trước."""
        key = self._key(user_id)
        items = self.client.lrange(key, 0, -1)
        return [json.loads(m) for m in items] if items else []

    def clear(self, user_id: str) -> None:
        self.client.delete(self._key(user_id))
```

#### File `app/lifecycle.py`
```python
"""CP4 — Graceful shutdown."""

from __future__ import annotations

import signal


class Lifecycle:
    """Giữ trạng thái vòng đời của process."""

    def __init__(self) -> None:
        self.shutting_down = False
        self._previous: dict = {}

    def request_shutdown(self, signum=None, frame=None) -> None:
        """Signal handler: đánh dấu process đang tắt dần."""
        self.shutting_down = True
        previous = self._previous.get(signum)
        if callable(previous):
            previous(signum, frame)

    def install(self) -> None:
        """Đăng ký handler cho SIGTERM và SIGINT, nhớ lại handler cũ."""
        for sig in (signal.SIGTERM, signal.SIGINT):
            self._previous[sig] = signal.getsignal(sig)
            signal.signal(sig, self.request_shutdown)


lifecycle = Lifecycle()
```

#### File `app/main.py` (Tích hợp toàn bộ CP1, CP3, CP4)
```python
"""Agent service — điểm ráp nối toàn bộ hệ thống."""

from __future__ import annotations

from contextlib import asynccontextmanager
from functools import lru_cache

from fastapi import Depends, FastAPI
from fastapi.responses import JSONResponse
from pydantic import BaseModel, Field

from utils.mock_llm import ask_llm

from .auth import verify_api_key
from .config import get_settings
from .cost_guard import CostGuard
from .lifecycle import lifecycle
from .logging_utils import log_event
from .rate_limiter import RateLimiter
from .store import ConversationStore, get_redis_client

SERVICE_NAME = "day12-agent"
SERVICE_VERSION = "1.0.0"


@lru_cache(maxsize=1)
def get_store() -> ConversationStore:
    return ConversationStore(get_redis_client())


@lru_cache(maxsize=1)
def get_rate_limiter() -> RateLimiter:
    return RateLimiter(get_redis_client(), get_settings().rate_limit_per_minute)


@lru_cache(maxsize=1)
def get_cost_guard() -> CostGuard:
    return CostGuard(get_redis_client(), get_settings().monthly_budget_usd)


@asynccontextmanager
async def lifespan(_app: FastAPI):
    lifecycle.install()
    log_event("service_started", service=SERVICE_NAME, version=SERVICE_VERSION)
    yield
    log_event("service_stopped", service=SERVICE_NAME)


app = FastAPI(title="Day 12 Production Agent", version=SERVICE_VERSION, lifespan=lifespan)


class AskRequest(BaseModel):
    question: str = Field(min_length=1, max_length=2000)


@app.get("/health")
def health():
    """Liveness probe — không kiểm tra Redis."""
    if lifecycle.shutting_down:
        return JSONResponse(status_code=503, content={"status": "shutting_down"})
    return {"status": "ok", "service": SERVICE_NAME, "version": SERVICE_VERSION}


@app.get("/ready")
def ready(store: ConversationStore = Depends(get_store)):
    """Readiness probe — kiểm tra kết nối Redis."""
    if lifecycle.shutting_down:
        return JSONResponse(status_code=503, content={"status": "shutting_down"})
    if not store.ping():
        return JSONResponse(status_code=503, content={"status": "not ready", "redis": False})
    return {"status": "ready", "redis": True}


@app.post("/ask")
def ask(
    payload: AskRequest,
    user_id: str = Depends(verify_api_key),
    store: ConversationStore = Depends(get_store),
    limiter: RateLimiter = Depends(get_rate_limiter),
    guard: CostGuard = Depends(get_cost_guard),
):
    """Hỏi agent một câu — thực thi theo thứ tự bảo vệ nghiêm ngặt."""
    limiter.check(user_id)
    guard.check(user_id)
    history = store.get_history(user_id)
    result = ask_llm(payload.question, history)
    store.append(user_id, "user", payload.question)
    store.append(user_id, "assistant", result["answer"])
    guard.record(user_id, result["cost_usd"])
    log_event(
        "ask_completed",
        user_id=user_id,
        tokens_in=result["tokens_in"],
        tokens_out=result["tokens_out"],
        cost_usd=result["cost_usd"],
    )
    return {
        "answer": result["answer"],
        "user_id": user_id,
        "history_length": len(history),
        "cost_usd": result["cost_usd"],
        "tokens": {"in": result["tokens_in"], "out": result["tokens_out"]},
    }


if __name__ == "__main__":
    import uvicorn

    settings = get_settings()
    uvicorn.run(app, host="0.0.0.0", port=settings.port)
```

---

## Bảng Tổng Hợp Kiểm Thử & Chấm Điểm

Chạy script chấm điểm tự động:
```bash
python grade.py --no-bonus
```

| Checkpoint | Nội dung đánh giá | Số Test | Kết quả | Điểm đạt được |
| :--- | :--- | :---: | :---: | :---: |
| **CP1** | 12-Factor Config, Health Check & Structured Logging | 13/13 | **PASSED** | **15.0 / 15.0** |
| **CP2** | Multi-stage Dockerfile, Non-root, Docker Compose | 14/14 | **PASSED** | **15.0 / 15.0** |
| **CP3** | API Key Timing Defense, Sliding Window, Cost Guard | 22/22 | **PASSED** | **20.0 / 20.0** |
| **CP4** | Stateless Redis Store, Readiness Probe, Graceful Shutdown | 19/19 | **PASSED** | **20.0 / 20.0** |
| **TỔNG LẬP TRÌNH LÕI** | **Toàn bộ 4 Checkpoint CP1 – CP4** | **68/68** | **PASSED (100%)** | **70.0 / 70.0** |
