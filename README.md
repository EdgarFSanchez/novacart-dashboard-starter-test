# NovaCart Account Dashboard
### HC&D Associates Capstone — App Developer + App Consultant

NovaCart is an account-management dashboard built with React and FastAPI. It displays order, product, customer, and geographic revenue data from either the included SQLite database or the NovaCart Snowflake Gold layer.

---

## What's in this repo

```
backend/          Python + FastAPI API
  main.py         ← Dashboard API endpoints
  connection.py   ← Handles SQLite, local Snowflake, and SPCS connections
  requirements.txt
  Dockerfile

frontend/         React 18 dashboard
  src/pages/
    OrdersView.js
    ProductsView.js
    CustomersView.js
  src/components/ ← Navbar and ServiceStatus
  src/utils/      ← API client and ThemeContext
  Dockerfile

router/           NGINX reverse proxy — do not modify
data/
  novacart_gold.db  ← SQLite database for local development
                      30,000 orders · 400 customers · 15 products

build-and-push.sh   ← Builds and pushes the three SPCS images
```

---

## Quick Start — Local Development

### 1. Backend

```bash
cd backend
cp .env.example .env
# No changes needed — DATA_BACKEND=sqlite works out of the box

pip install -r requirements.txt
uvicorn main:app --reload --port 8000
```

Open **http://localhost:8000/docs** — Swagger UI with all endpoints.

Test the health endpoint:
```bash
curl http://localhost:8000/health
```

### 2. Frontend

```bash
cd frontend
cp .env.example .env

npm install
npm start
# Opens at http://localhost:3000
```

On Windows PowerShell, use `npm.cmd start` if PowerShell blocks `npm.ps1`.

### Optional — Use Snowflake Locally

The backend supports Snowflake key-pair authentication. Each developer should use a private key registered for the assigned Snowflake user. Never commit a private key or `.env` file.

Update `backend/.env`:

```dotenv
DATA_BACKEND=snowflake
CLIENT_VALIDATION=Dev

SNOWFLAKE_ACCOUNT=<account_identifier>
SNOWFLAKE_USERNAME=<username>
SNOWFLAKE_ROLE=<assigned_role>
SNOWFLAKE_WAREHOUSE=NOVACART_APP_WH
SNOWFLAKE_DATABASE=NOVACART_DB
SNOWFLAKE_SCHEMA=APP
SNOWFLAKE_PRIVATE_KEY_PATH=<absolute_path_to_rsa_key.p8>

PORT=8000
```

After starting the backend, open **http://localhost:8000/health**. A successful connection reports `"backend": "snowflake"` and `"database": {"status": "connected"}`.

---

## Implemented Functionality

### Backend API

The FastAPI backend provides these dashboard endpoints:

| Endpoint | Description |
|---|---|
| `GET /franchise/summary` | Total revenue, orders, unique customers |
| `GET /franchise/orders` | Monthly order volume and revenue |
| `GET /franchise/products` | Top 10 products by revenue |
| `GET /franchise/customers` | Top 20 customers by revenue |
| `GET /franchise/cities` | Revenue by city and state |

The date-filtered endpoints accept `start` and `end` query parameters in `YYYY-MM-DD` format. Revenue calculations include orders with `status IN ('delivered', 'shipped')`.

### Frontend Dashboard

| File | Functionality |
|---|---|
| `OrdersView.js` | Stat cards + monthly revenue chart + cities chart |
| `ProductsView.js` | Products bar chart + products table |
| `CustomersView.js` | Sortable customers table |

The dashboard also includes date filters, light and dark themes, and a backend service-status indicator.

### Project Deliverables

- Write the requirements document before any code is written (end of Day 1)
- Validate each endpoint against requirements before marking it done
- Write the Solution Design Document by Day 4
- Prepare and lead the client presentation on Day 5

---

## Data Schema

The SQLite database has four tables matching the Gold layer from the Data Engineering capstone:

```
fact_orders    order_id, customer_id, product_id, order_date, amount,
               currency, status, quantity, date_key

dim_customer   customer_id, name, email, signup_date,
               addr_street, addr_city, addr_state, addr_zip,
               valid_from, valid_to, is_current

dim_product    product_id, name, category, price, updated_at

dim_date       date_key, full_date, year, quarter, month,
               month_name, day_of_week, is_weekend
```

Use `status IN ('delivered', 'shipped')` for revenue calculations.

---

## Deploying to SPCS

The deployment uses three Docker images:

- `backend_service_image_group<GROUP>`
- `frontend_service_image_group<GROUP>`
- `router_service_image_group<GROUP>`

The `build-and-push.sh` script builds and pushes these images to the Snowflake Image Repository. It does not create the Snowpark Container Services service. The facilitator or the **Deploy Group** GitHub Actions workflow performs that step.

### Prerequisites

- Docker Desktop installed and running
- Snowflake CLI installed
- A Snowflake CLI connection named `spcs`
- Access to the project Snowflake Image Repository
- Your assigned group number
- The latest code from `main`

Verify the tools and Snowflake connection:

```bash
docker version
snow --version
snow connection test --connection spcs
```

Update and verify the local repository:

```bash
git switch main
git pull --ff-only origin main
git status
```

The working tree should be clean before building.

### Set Deployment Variables

The facilitator provides the image repository URL and group number. Do not include a trailing slash in `REPO_URL`.

#### macOS or Linux

```bash
export REPO_URL="<snowflake_image_repository_url>"
export GROUP="<group_number>"
```

#### Windows PowerShell

```powershell
$env:REPO_URL = "<snowflake_image_repository_url>"
$env:GROUP = "<group_number>"
```

### Build and Push

From the repository root on macOS or Linux:

```bash
bash build-and-push.sh
```

On Windows, run the script through Git Bash:

```powershell
& "C:\Program Files\Git\bin\bash.exe" build-and-push.sh
```

The script:

1. Logs in to the Snowflake image registry.
2. Builds the backend image.
3. Builds the frontend image.
4. Builds the NGINX router image.
5. Tags and pushes all three images.

Successful output ends with:

```text
All images pushed. Notify your facilitator
so they can create your SPCS services.
```

### Deploy the Service

After all images are pushed, provide the facilitator with:

- Your group number
- The Git commit being deployed
- Confirmation that all three images were pushed

The facilitator can deploy the service manually or run the repository's **Deploy Group** GitHub Actions workflow. In SPCS, Snowflake mounts an OAuth token for the backend automatically; local `.env` files and private keys must not be included in the images.

### Verify the Deployment

Open the public URL provided by the facilitator and verify:

- The Orders page displays summary cards and both charts.
- The Products page displays its chart and table.
- The Customers page displays its sortable table.
- Date filters work on every page.
- The service indicator reports **Service healthy**.
- Browser developer tools show no failed API requests.

The deployed backend health response should report `"backend": "snowflake"` and a connected database.

---

## Troubleshooting

**Backend can't find the database** — Run `uvicorn` from inside the `backend/` directory.

**CORS error or `Failed to fetch` in the browser** — Make sure `CLIENT_VALIDATION=Dev` is set in `backend/.env` and open the frontend at `http://localhost:3000`.

**Snowflake reports `JWT token is invalid`** — Confirm the account and username, use an absolute private-key path, verify that the matching public key is registered for the Snowflake user, and check that the computer clock is synchronized.

**`snow` command not found** — Run:
```bash
pip install snowflake-cli-labs
```

Restart the terminal after installation.

**PowerShell blocks `npm.ps1`** — Use `npm.cmd start` from the `frontend` directory.

**Docker build fails** — Confirm Docker Desktop is running and has sufficient disk space. The deployment script already uses `--no-cache` and targets `linux/amd64`.
