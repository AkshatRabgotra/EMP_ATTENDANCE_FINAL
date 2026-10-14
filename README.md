# Employee Attendance & Analytics API

FastAPI + MongoDB backend service implementing employee management, attendance lifecycle (punch-in, punch-out, regularization), and high-performance analytical aggregation pipelines.

---

## 1. Quick Start

### Requirements
- Python 3.11+
- MongoDB 6.0+ (Atlas M0 free tier or local Docker container)

### Setup & Run
```bash
# 1. Install dependencies
pip install -r requirements.txt

# 2. Configure environment (optional, defaults provided)
# MONGO_URI defaults to "mongodb://localhost:27017"
# MONGO_DB defaults to "attendance_db"
export MONGO_URI="mongodb://localhost:27017"
export MONGO_DB="attendance_db"

# 3. Start the application
uvicorn app.main:app --port 8000
```

The application creates all required database indexes idempotently at startup during the FastAPI lifespan.
- **Interactive OpenAPI Documentation / Swagger UI:** `http://localhost:8000/docs`
- **ReDoc Documentation:** `http://localhost:8000/redoc`
- **Health Check Probe:** `GET http://localhost:8000/health` (returns HTTP 200 when MongoDB is connected, 503 otherwise).

---

## 2. Key Architecture & Design Choices

- **Single-File Architecture:** All application code, models, pure rule calculators, serializers, and aggregation pipelines are encapsulated in `app/main.py`.
- **Timezone Awareness & Instants:** All database instants are stored as timezone-aware UTC BSON datetimes. API instants are validated as strict integer epoch milliseconds (`StrictInt`, range `100000000000 <= ms <= 4102444800000`) and truncated to whole seconds before calculation and storage.
- **Decimal Precision (R4, R8):** All durations, rates, and percentages are computed with Python's `Decimal` module and quantized using `ROUND_HALF_UP` to prevent floating-point and banker's rounding errors.
- **Race Safety & Atomic Writes:**
  - Punch-in is guarded by a unique compound index on `(emp_code, date)`. Duplicate concurrent requests result in HTTP 409 Conflict.
  - Punch-out uses conditional atomic updates (`update_one({"_id": rec_id, "punch_out": None}, ...)`) so concurrent calls produce exactly one HTTP 200.
  - Regularization employs optimistic concurrency checking the history array length to guarantee zero lost audit entries.
- **Database-Driven Analytics:**
  - Analytics pipelines never fetch entire collections into Python memory.
  - Leaderboard uses MongoDB `$setWindowFields` with `$rank` for competition ranking (1, 2, 2, 4).
  - Trend uses `$densify` with 1-day stepping across calendar dates, with a `$unionWith` anchor guaranteeing complete date ranges even for empty datasets.

---

## 3. Explicit Assumptions & Ambiguity Resolutions

1. **Trend Moving Average:** The 7-day moving average averages the reported 4-decimal rounded `attendance_rate` values among the current row and up to 6 preceding rows within the requested range (ignoring nulls), rounded to 4 decimals half-up.
2. **Trend Date Range Bound:** Exactly 92 inclusive calendar days (`(to - from).days + 1 <= 92`) is allowed. Any span exceeding 92 days returns HTTP 422.
3. **Punch-Out Condition Precedence:** If multiple conditions could apply, status evaluation proceeds in order: `404 Not Found` (unknown employee or no open record) $\to$ `409 Conflict` (record already punched out) $\to$ `422 Unprocessable Entity` (punch-out timestamp is $\le$ punch-in or $> 24$ hours).
4. **Regularization No-Op:** If a `PATCH` request provides values that match the currently stored state exactly (resulting in an empty `changes` set), it returns `HTTP 422 Unprocessable Entity` ("No fields were changed").
5. **Leaderboard Tie Slicing:** When employees tie at a rank $\le \text{limit}$, all tied employees are returned. The response array may therefore contain more than `limit` items.
