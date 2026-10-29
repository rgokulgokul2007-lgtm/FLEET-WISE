# FleetWise — Vehicle & Delivery Assignment

FleetWise is a production-quality, commercial fleet dispatch and delivery assignment web application built for **IT HAPPENS @ RAALE #13**.

The application replaces manual dispatch spreadsheets with an automated, centralized dispatch system that pairs pending deliveries with eligible vehicles and drivers, enforcing strict resource exclusivity so that **no vehicle or driver can ever be double-booked across active deliveries**.

---

## Key Features

- **Strict Light-Themed Commercial UI**: Professional blue and neutral light grey styling with accessible contrast, intuitive icons, interactive status badges, and mobile-friendly responsive navigation.
- **Atomic Dispatch Assignment**: 1-click resource assignment pairing 1 eligible vehicle and 1 eligible driver to a pending delivery inside an immediate database transaction (`BEGIN IMMEDIATE`).
- **Database-Enforced Exclusivity & Concurrency Protection**:
  - Unique partial indexes enforce at the database engine level that at most one active assignment exists per vehicle and driver:
    ```sql
    CREATE UNIQUE INDEX idx_active_assignment_vehicle ON assignments(vehicle_id) WHERE is_active = 1;
    CREATE UNIQUE INDEX idx_active_assignment_driver ON assignments(driver_id) WHERE is_active = 1;
    CREATE UNIQUE INDEX idx_active_assignment_delivery ON assignments(delivery_id) WHERE is_active = 1;
    ```
  - Simultaneous conflicting requests result in exactly one successful assignment (HTTP 201) and immediate rejection of conflicting attempts (HTTP 409) with no partial state updates.
- **Delivery Lifecycle Engine**: Full state machine (`pending` → `assigned` → `in_transit` → `delivered`), with safe cancellation and automatic resource release.
- **Vehicle & Driver CRUD with Historical Protection**: Active assignments block status changes to maintenance/inactive/unavailable or deletions until resolved.
- **Real-Time Operational Dashboard**: Real database metrics (not hardcoded), fleet utilization percentage, live resource queues, and 1-click demo seeding.
- **Comprehensive Audit Trail & CSV Export**: All state transitions and dispatches are logged to an immutable activity ledger. Raw CSV exports are available for Deliveries, Vehicles, Drivers, and Assignments.
- **12 Automated Acceptance Tests**: Complete unit and integration test suite executing cases T1 through T12.

---

## Tech Stack

| Layer | Technology |
|---|---|
| **Frontend** | React 19, TypeScript, Tailwind CSS, Vite, Lucide Icons |
| **Backend** | Node.js, Express 5, TypeScript, Zod |
| **Database** | SQLite via `better-sqlite3` (WAL mode, Foreign Keys ON, Immediate Transactions) |
| **Testing** | Vitest, Supertest |

---

## Installation & Startup Instructions

### Prerequisites
- Node.js (v18.x or newer, tested on Node v24)
- npm (v9.x or newer)

### 1. Clone & Install Dependencies

From the project root:
```bash
# Install root orchestration tools
npm install

# Install backend dependencies
cd backend
npm install

# Install frontend dependencies
cd ../frontend
npm install

# Return to root
cd ..
```

### 2. Environment Configuration

Copy `.env.example` in both root and `backend`:
```bash
cp .env.example backend/.env
```
Default configuration:
```env
PORT=5000
NODE_ENV=development
DATABASE_PATH=./data/fleetwise.db
CORS_ORIGIN=http://localhost:5173
```

### 3. Run Development Servers

From the root directory, run both frontend and backend concurrently:
```bash
npm run dev
```

Alternatively, run in separate terminal tabs:
```bash
# Terminal 1: Backend API (port 5000)
cd backend
npm run dev

# Terminal 2: Frontend Client (port 5173)
cd frontend
npm run dev
```

Open your browser to: **http://localhost:5173**  
API Health Check: **http://localhost:5000/health**

---

## Automated Acceptance Testing (T1 - T12)

FleetWise includes an automated test suite executing all twelve mandatory test cases defined in the project specification.

To run the automated tests:
```bash
# From root
npm run test

# Or directly in backend
cd backend
npm run test
```

### Acceptance Test Matrix

| Test ID | Scenario | Expected Result | Result |
|---|---|---|---|
| **T1** | Valid Assignment | Available vehicle + available driver assigned to pending delivery; related records updated atomically | **PASS** |
| **T2** | Vehicle Conflict | Attempt to assign occupied vehicle to another delivery rejected (HTTP 409); no partial update | **PASS** |
| **T3** | Driver Conflict | Attempt to assign occupied driver to another delivery rejected (HTTP 409); no partial update | **PASS** |
| **T4** | Both Resources Occupied | Attempt to assign occupied vehicle and driver rejected with clear error (HTTP 409) | **PASS** |
| **T5** | Unavailable Driver | Attempt assignment with unavailable driver rejected (HTTP 409) | **PASS** |
| **T6** | Maintenance Vehicle | Attempt assignment with vehicle under maintenance rejected (HTTP 409) | **PASS** |
| **T7** | Unknown ID | Assignment request with unknown vehicle, driver, or delivery rejected safely (HTTP 404) | **PASS** |
| **T8** | Complete Delivery | Delivered delivery closes active assignment and safely releases vehicle and driver to available | **PASS** |
| **T9** | Cancel Delivery | Cancelled assigned delivery releases vehicle and driver; records marked cancelled | **PASS** |
| **T10** | Invalid Transition | Direct pending → delivered transition rejected (HTTP 400) | **PASS** |
| **T11** | Duplicate Submission | Rapid repeated assignment submission rejected (HTTP 409); exactly 1 assignment created | **PASS** |
| **T12** | Concurrent Requests | Parallel competing requests for same resource results in exactly 1 success (201) and 1 conflict (409) | **PASS** |

---

## API Documentation

### Vehicles
- `GET /vehicles`: List all vehicles with active delivery metadata. Filters: `status`, `type`, `search`.
- `GET /vehicles/:id`: Retrieve single vehicle details.
- `GET /vehicles/:id/history`: Assignment history for this vehicle.
- `POST /vehicles`: Register vehicle (`vehicle_id`, `registration_no`, `type`, `capacity`, `status`).
- `PATCH /vehicles/:id`: Update vehicle details. Blocked if actively assigned to delivery.
- `DELETE /vehicles/:id`: Safe removal or archival.

### Drivers
- `GET /drivers`: List all drivers with active delivery metadata. Filters: `availability_status`, `search`.
- `GET /drivers/:id`: Retrieve single driver details.
- `GET /drivers/:id/history`: Assignment history for this driver.
- `POST /drivers`: Register driver (`driver_id`, `name`, `phone`, `availability_status`).
- `PATCH /drivers/:id`: Update driver details. Blocked from unavailable status if assigned.
- `DELETE /drivers/:id`: Safe removal or archival.

### Deliveries
- `GET /deliveries`: List all delivery orders. Filters: `status`, `priority`, `vehicle_id`, `driver_id`, `search`, `sortBy`, `sortOrder`.
- `GET /deliveries/:id`: Retrieve single delivery with assigned vehicle and driver.
- `GET /deliveries/:id/history`: Complete assignment history and audit logs.
- `POST /deliveries`: Create new delivery in `pending` state.
- `PATCH /deliveries/:id`: Edit permitted details (pickup, dropoff, scheduled time, priority).
- `POST /deliveries/:id/assign`: **Atomic dispatch assignment** (`vehicle_id`, `driver_id`).
- `PATCH /deliveries/:id/status`: Transition lifecycle status (`in_transit`, `delivered`, `cancelled`).
- `DELETE /deliveries/:id`: Safe removal.

### Assignments & System
- `GET /assignments`: Query assignment ledger with filters (`is_active`, `search`, `vehicle_id`, `driver_id`).
- `GET /reports/metrics`: Live dashboard operational metrics.
- `GET /reports/operational`: Breakdown by vehicle type and delivery priority.
- `GET /reports/logs`: Chronological audit activity logs.
- `GET /reports/export/:type`: Download raw CSV file (`deliveries`, `vehicles`, `drivers`, `assignments`).
- `GET /health`: Engine heartbeat, database status, memory RSS/heap, table counts.
- `POST /health/seed`: Seed demo fleet dataset.
- `POST /health/reset`: Clear all tables to clean state.
