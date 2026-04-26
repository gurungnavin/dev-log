# Square Samurai Test Backend

Standalone FastAPI backend for testing **Square Sandbox OAuth** and Square APIs before merging into the real Samurai Tax backend.

This README also includes the troubleshooting log from our setup so we do not repeat the same problems later.

---

## 1. Purpose

We are building this as a **standalone test backend** first.

The goal is to confirm this flow:

```txt
Merchant / Postman / Frontend
→ Samurai Test Backend
→ Square Sandbox OAuth
→ Square Sandbox APIs
→ PostgreSQL
```

Important rule:

```txt
Frontend should not call Square directly.
Frontend or Postman calls our backend.
Backend calls Square and stores Square access tokens.
```

---

## 2. Project Name

```txt
square-samurai-test-backend
```

---

## 3. Stack

- Python
- FastAPI
- Uvicorn
- PostgreSQL
- Docker Compose
- SQLModel / SQLAlchemy
- requests
- Postman
- Square Sandbox only for now

---

## 4. Ports

```txt
Backend inside Docker: 8000
Backend on host/Mac:   8800

PostgreSQL inside Docker: 5432
PostgreSQL on host/Mac:   5544
```

So the backend is accessed from browser/Postman with:

```txt
http://localhost:8800
```

---

## 5. Folder Structure

```txt
square-samurai-test-backend/
├── Dockerfile
├── README.md
├── .env.sample
├── app
│   ├── __init__.py
│   ├── config.py
│   ├── db
│   │   ├── __init__.py
│   │   ├── init_db.py
│   │   └── session.py
│   ├── main.py
│   ├── models
│   │   ├── __init__.py
│   │   ├── receipt_transaction.py
│   │   └── square_connection.py
│   ├── routers
│   │   ├── __init__.py
│   │   └── square.py
│   ├── schemas
│   │   ├── __init__.py
│   │   ├── receipt.py
│   │   └── square.py
│   └── services
│       ├── __init__.py
│       ├── square_client.py
│       └── square_oauth.py
├── docker-compose.yml
└── requirements.txt
```

Important:

```txt
Do NOT create app/database.py
Do NOT create app/models.py
```

Use:

```txt
app/db/session.py
app/db/init_db.py
app/models/square_connection.py
app/services/square_oauth.py
app/services/square_client.py
app/routers/square.py
```

---

## 6. Environment Variables

Create `.env` from `.env.sample`.

```bash
cp .env.sample .env
```

Example `.env.sample`:

```env
APP_NAME=Square Samurai Test Backend
DEBUG=true

DATABASE_URL=postgresql://postgres:postgres@postgres:5432/square_samurai_test

SQUARE_ENVIRONMENT=sandbox
SQUARE_BASE_URL=https://connect.squareupsandbox.com

SQUARE_APPLICATION_ID=
SQUARE_APPLICATION_SECRET=

SQUARE_REDIRECT_URI=http://localhost:8800/api/square/oauth/callback
SQUARE_VERSION=2026-01-22
SQUARE_SCOPES=MERCHANT_PROFILE_READ PAYMENTS_READ ORDERS_READ
```

Security note:

```txt
Never commit real .env to GitHub.
Never share SQUARE_APPLICATION_SECRET in chat, screenshots, or GitHub.
```

Use `.gitignore`:

```gitignore
.env
.venv/
__pycache__/
*.pyc
```

---

## 7. Square Developer Console Setup

Open:

```txt
Square Developer Console
→ Applications
→ Open your test application
→ Sandbox environment
```

Copy:

```txt
Sandbox Application ID
Sandbox Application Secret
```

Then paste into `.env`:

```env
SQUARE_APPLICATION_ID=sandbox-sq0idb-xxxxxxxxxxxxxxxx
SQUARE_APPLICATION_SECRET=xxxxxxxxxxxxxxxx
```

Add the OAuth redirect URL in the Square Developer Console:

```txt
http://localhost:8800/api/square/oauth/callback
```

It must match exactly.

Correct:

```txt
http://localhost:8800/api/square/oauth/callback
```

Incorrect:

```txt
http://localhost:8800/api/square/oauth/callback/
```

No trailing slash.

---

## 8. Run Backend

```bash
docker compose down
docker compose up --build
```

Health check:

```bash
curl http://localhost:8800/health
```

Expected:

```json
{
  "status": "ok",
  "app": "Square Samurai Test Backend",
  "environment": "sandbox"
}
```

---

## 9. OAuth Test Flow

### Step 1: Generate OAuth URL

```bash
curl http://localhost:8800/api/square/oauth/url
```

Expected response:

```json
{
  "auth_url": "https://connect.squareupsandbox.com/oauth2/authorize?client_id=sandbox-sq0idb-...&scope=MERCHANT_PROFILE_READ+PAYMENTS_READ+ORDERS_READ&state=...",
  "state": "..."
}
```

Important:

```txt
client_id must NOT be empty.
```

Bad:

```txt
client_id=
```

Good:

```txt
client_id=sandbox-sq0idb-xxxxxxxx
```

---

### Step 2: Open OAuth URL

Copy `auth_url` and open it in browser.

For Sandbox testing, if the OAuth page is blank or returns 400, first open the Sandbox test seller dashboard.

Recommended Sandbox flow:

```txt
1. Square Developer Console
2. Sandbox test accounts
3. Open one test account / seller dashboard
4. In the same browser session, open the OAuth URL
5. Square permission screen should appear
```

This is only a Sandbox testing workaround.

Production users should NOT need this.

---

### Step 3: Approve Permission

On the Square OAuth page, click:

```txt
Allow
```

Square should redirect to:

```txt
http://localhost:8800/api/square/oauth/callback?code=...&state=...
```

Expected backend response:

```json
{
  "message": "Square Sandbox connected successfully",
  "merchant_id": "...",
  "merchant_name": "...",
  "environment": "sandbox",
  "scope": "MERCHANT_PROFILE_READ PAYMENTS_READ ORDERS_READ",
  "next_steps": [
    "GET /api/square/locations/...",
    "GET /api/square/payments/...?location_id=YOUR_LOCATION_ID"
  ]
}
```

---

## 10. API Endpoints

```txt
GET /health

GET /api/square/oauth/url
GET /api/square/oauth/callback

GET /api/square/connections
GET /api/square/merchant/{merchant_id}
GET /api/square/locations/{merchant_id}
GET /api/square/payments/{merchant_id}?location_id={location_id}
GET /api/square/orders/{merchant_id}/{order_id}
```

---

## 11. Postman / Curl Test Order

### 1. Health

```bash
curl http://localhost:8800/health
```

### 2. OAuth URL

```bash
curl http://localhost:8800/api/square/oauth/url
```

### 3. Open `auth_url` in browser and click Allow

### 4. Check stored connections

```bash
curl http://localhost:8800/api/square/connections
```

### 5. Fetch merchant profile

```bash
curl http://localhost:8800/api/square/merchant/YOUR_MERCHANT_ID
```

### 6. Fetch Square locations / branches

```bash
curl http://localhost:8800/api/square/locations/YOUR_MERCHANT_ID
```

Copy one `location_id`.

### 7. Fetch payments for selected location

```bash
curl "http://localhost:8800/api/square/payments/YOUR_MERCHANT_ID?location_id=YOUR_LOCATION_ID"
```

Optional date filter:

```bash
curl "http://localhost:8800/api/square/payments/YOUR_MERCHANT_ID?location_id=YOUR_LOCATION_ID&begin_time=2026-04-01T00:00:00Z&end_time=2026-04-30T23:59:59Z"
```

### 8. Fetch related order

Use `order_id` from the payment response:

```bash
curl http://localhost:8800/api/square/orders/YOUR_MERCHANT_ID/YOUR_ORDER_ID
```

---

## 12. Database Tables

Auto-created on startup for this test backend:

```txt
square_connections
square_locations
square_payments
square_orders
```

We are storing:

```txt
normalized fields + raw Square JSON
```

Reason:

```txt
Final Samurai Tax required fields are not decided yet,
so raw Square JSON protects us from losing useful data.
```

Later, when merging into the real backend:

```txt
Replace auto-create tables with Alembic migrations.
```

---

## 13. PostgreSQL Check

Enter PostgreSQL container:

```bash
docker exec -it square_samurai_test_postgres psql -U postgres -d square_samurai_test
```

Show tables:

```sql
\dt
```

Check data:

```sql
SELECT id, merchant_id, merchant_name, environment, created_at
FROM square_connections;

SELECT id, merchant_id, location_id, name, currency, timezone
FROM square_locations;

SELECT id, merchant_id, location_id, payment_id, order_id, amount, currency
FROM square_payments;

SELECT id, merchant_id, location_id, order_id, total_amount, currency
FROM square_orders;
```

---

## 14. pgAdmin Connection

Use:

```txt
Host: localhost
Port: 5544
Database: square_samurai_test
Username: postgres
Password: postgres
```

---

# Troubleshooting Log

This section records problems we already faced.

---

## Problem 1: OAuth URL had empty client_id

### Symptom

```json
{
  "auth_url": "https://connect.squareupsandbox.com/oauth2/authorize?client_id=&scope=..."
}
```

### Cause

`.env` was missing:

```env
SQUARE_APPLICATION_ID=
```

or Docker had not restarted after `.env` update.

### Fix

Update `.env`:

```env
SQUARE_APPLICATION_ID=sandbox-sq0idb-xxxxxxxx
SQUARE_APPLICATION_SECRET=xxxxxxxx
```

Restart Docker:

```bash
docker compose down
docker compose up --build
```

Verify Docker is reading env:

```bash
docker compose exec backend printenv SQUARE_APPLICATION_ID
docker compose exec backend printenv SQUARE_REDIRECT_URI
```

Expected:

```txt
sandbox-sq0idb-xxxxxxxx
http://localhost:8800/api/square/oauth/callback
```

---

## Problem 2: Square OAuth page blank or 400

### Symptom

Browser shows blank page or console/network shows 400 when opening:

```txt
https://connect.squareupsandbox.com/oauth2/authorize?client_id=...
```

### Cause

Sandbox OAuth may need a Sandbox seller session first.

Opening the OAuth URL while only logged into the developer account can fail.

### Fix

Use this order:

```txt
1. Open Square Developer Console
2. Go to Sandbox test accounts
3. Open one Sandbox seller dashboard
4. In the same browser session, open the OAuth URL
5. Permission screen appears
```

Incognito can also help:

```txt
1. Open Incognito
2. Login to Square Developer
3. Open Sandbox test account seller dashboard
4. Open OAuth URL
```

Important:

```txt
This is only for Sandbox testing.
Production users should not need to open the dashboard.
```

---

## Problem 3: Confusion about Greenback-style OAuth

### Concern

Greenback/Dext Commerce does not require users to open a dashboard first.

### Explanation

Correct.

Greenback is a production app. Real merchants use production Square OAuth:

```txt
Merchant clicks Connect
→ Square login/permission screen
→ Merchant clicks Allow
→ App receives OAuth callback
```

Our current flow is Sandbox-only:

```txt
Developer opens Sandbox seller dashboard first
→ Then opens OAuth URL
→ Permission screen appears
```

Production Samurai Tax should behave like Greenback:

```txt
Store owner opens Samurai Tax / Square Marketplace
→ Clicks Connect with Square
→ Logs into Square
→ Clicks Allow
→ Samurai backend stores token
→ Store selects Square location/branch
```

No production user should open:

```txt
Square Developer Console
Sandbox test account
Sandbox seller dashboard
```

---

## Problem 4: Pylance `dict[Unknown, Unknown]` errors in `routers/square.py`

### Symptom

Pylance errors like:

```txt
Type of parameter "token_json" is partially unknown
Expected type arguments for generic class "dict"
Type of "get" is partially unknown
```

### Cause

Square API responses are JSON dictionaries, but Python typing saw them as raw `dict`.

### Fix

Use:

```python
from typing import Any, Optional, cast

JsonDict = dict[str, Any]
```

and typed helper functions:

```python
def get_str(data: JsonDict, key: str) -> Optional[str]:
    value: object = data.get(key)
    return value if isinstance(value, str) else None


def get_required_str(data: JsonDict, key: str) -> str:
    value: object = data.get(key)

    if not isinstance(value, str) or not value:
        raise HTTPException(
            status_code=400,
            detail=f"Missing required Square field: {key}",
        )

    return value


def get_dict(data: JsonDict, key: str) -> JsonDict:
    value: object = data.get(key)

    if isinstance(value, dict):
        return cast(JsonDict, value)

    return {}
```

---

## Problem 5: `TypeGuard` unknown import symbol

### Symptom

Pylance showed:

```txt
"TypeGuard" is unknown import symbol
```

### Cause

VS Code/Pylance interpreter did not recognize `TypeGuard` from `typing`.

This can happen if VS Code is using a different Python interpreter than Docker.

### Fix

Avoid `TypeGuard`.

Use:

```python
from typing import Any, Optional, cast
```

Do not use:

```python
from typing import TypeGuard
```

---

## Problem 6: OAuth URL contained `session=false`

### Symptom

Square permission screen showed a note:

```txt
While you test in sandbox mode the session parameter will not force logout before viewing this screen.
```

### Explanation

For Sandbox, `session` does not force logout.

### Recommended Fix

In `app/services/square_oauth.py`, use:

```python
params = {
    "client_id": settings.SQUARE_APPLICATION_ID,
    "scope": settings.SQUARE_SCOPES,
    "state": state,
}
```

Avoid:

```python
params = {
    "client_id": settings.SQUARE_APPLICATION_ID,
    "scope": settings.SQUARE_SCOPES,
    "session": "false",
    "state": state,
}
```

---

## 15. Current Square Scopes

```txt
MERCHANT_PROFILE_READ
PAYMENTS_READ
ORDERS_READ
```

Meaning:

```txt
MERCHANT_PROFILE_READ → merchant profile + locations
PAYMENTS_READ         → payment history
ORDERS_READ           → Square order details
```

---

## 16. Production Target Flow

Final Samurai Tax production UX should be simple:

```txt
Store owner visits Samurai Tax or Square Marketplace
→ Clicks Connect with Square
→ Square login/permission page opens
→ Store owner clicks Allow
→ Square redirects to Samurai Tax backend
→ Backend stores token
→ Samurai dashboard displays locations/branches
→ Store selects branch/location
→ Samurai backend fetches payments/orders for selected location
→ Data continues into Samurai Tax Free workflow
```

Production users should not know about:

```txt
Square Developer Console
Sandbox test accounts
Sandbox seller dashboard
```

---

## 17. Next Development Steps

After OAuth connection succeeds:

```txt
1. Confirm square_connections row is saved
2. Fetch merchant profile
3. Fetch locations
4. Select one location_id
5. Fetch payments by location_id
6. Fetch order by order_id
7. Save normalized payment/order data
8. Decide which Square fields are needed for Samurai Tax transaction mapping
```

Later:

```txt
1. Add refresh token handling
2. Add token expiration check
3. Add location selection UI
4. Add payment/order sync job
5. Convert auto-create DB tables to Alembic migrations
6. Merge into real Samurai Tax backend
```
