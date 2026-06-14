# Boutique Microservices — Project Summary

## What Is This Project?

This is a **fake online fashion store** (think: a simplified version of ASOS or Net-a-Porter) built to demonstrate how modern cloud applications are structured. It sells things like silk evening gowns, designer bags, and accessories.

The key thing that makes it interesting is that it's built as **microservices** — instead of one big program doing everything, it's broken into 8+ small programs (services), each responsible for one job. They talk to each other over the network, just like real services at companies like Netflix or Uber.

---

## What Can the App Do?

- Browse and search fashion products
- Register and log in as a customer
- Place and view orders
- Admin users can manage the store

---

## The Building Blocks

### What is Docker?
Think of Docker as a way to package each service into its own sealed box (called a **container**). Each box has everything the service needs to run — code, dependencies, settings. This means the app runs the same way on any computer.

### What is Docker Compose?
Docker Compose is a tool that reads a single file (`docker-compose.yml`) and starts all the containers at once, in the right order, and connects them together.

---

## All the Services (The 10 Containers)

| # | Container Name | What It Does | Port |
|---|---|---|---|
| 1 | `boutique-frontend` | The website users see in their browser | 3000 |
| 2 | `boutique-gateway` | The front door — routes requests to the right service | 3001 |
| 3 | `boutique-auth` | Handles login, registration, and passwords | 3002 |
| 4 | `boutique-products` | Manages the product catalog (items, categories, images) | 3003 |
| 5 | `boutique-orders` | Handles creating new orders and checking stock/auth | 3004 |
| 6 | `boutique-order-management` | Read/manage existing orders (admin-style) | 3005 |
| 7 | `boutique-users` | Stores user profile information | 3006 |
| 8 | `boutique-grafana` | Dashboard to visually monitor the app's health | 3007 |
| 9 | `boutique-prometheus` | Collects health/performance numbers from every service | 9090 |
| 10 | `boutique-postgres` | The database — stores all persistent data | 5432 |

---

## Service Connection Map

This shows how the services talk to each other. Read it top-to-bottom — the user's browser is at the top, the database is at the bottom.

```
┌─────────────────────────────────────────────────────────────────┐
│                      YOUR BROWSER                               │
│                  http://localhost:3000                          │
└──────────────────────────┬──────────────────────────────────────┘
                           │ visits the site
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                   FRONTEND (nginx)                              │
│               boutique-frontend : port 3000                    │
│  Serves the React website. Static HTML/CSS/JS files.           │
│  All API calls go to the Gateway.                              │
└──────────────────────────┬──────────────────────────────────────┘
                           │ /api/* requests
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                    API GATEWAY                                  │
│               boutique-gateway : port 3001                     │
│  The single entry point for all backend traffic.               │
│  It reads the URL path and forwards to the right service:      │
│    /api/auth     → Auth Service                                │
│    /api/products → Product Service                             │
│    /api/orders   → Order Service                               │
│    /api/users    → User Service                                │
└────┬─────────────┬──────────────┬──────────────┬───────────────┘
     │             │              │              │
     ▼             ▼              ▼              ▼
┌─────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│  AUTH   │  │ PRODUCTS │  │  ORDER   │  │  USERS   │
│ :3002   │  │  :3003   │  │ SERVICE  │  │  :3006   │
│         │  │          │  │  :3004   │  │          │
│ Login   │  │ Product  │  │ Create   │  │ Profile  │
│ Register│  │ catalog  │  │ orders   │  │ info     │
│ JWT     │  │ categories│ │ validate │  │          │
│ tokens  │  │ images   │  │ stock    │  │          │
└────┬────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘
     │            │              │  │           │
     │            │   calls auth ◄──┘           │
     │            │   calls products◄────────────┘ (to check stock)
     │            │              │
     ▼            ▼              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   ORDER MANAGEMENT                              │
│               boutique-order-management : port 3005            │
│  A separate service for reading/managing placed orders.        │
│  Shares the same orders_db database as Order Service.          │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                     POSTGRESQL DATABASE                         │
│                  boutique-postgres : port 5432                 │
│                                                                 │
│  Four separate logical databases inside one PostgreSQL server: │
│                                                                 │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────────────┐ │
│  │  auth_db    │  │ products_db  │  │       orders_db        │ │
│  │─────────────│  │──────────────│  │────────────────────────│ │
│  │ users       │  │ categories   │  │ orders                 │ │
│  │  (login     │  │ products     │  │ order_items            │ │
│  │   data)     │  │ product_     │  │                        │ │
│  │             │  │ images       │  │ (shared by order-      │ │
│  └─────────────┘  └──────────────┘  │  service AND orders)   │ │
│                                     └────────────────────────┘ │
│  ┌─────────────┐                                               │
│  │  users_db   │                                               │
│  │─────────────│                                               │
│  │ user        │                                               │
│  │  profiles   │                                               │
│  └─────────────┘                                               │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    MONITORING STACK                             │
│                                                                 │
│  PROMETHEUS (port 9090)                                        │
│  Scrapes metrics from every service every 15 seconds:          │
│    gateway, auth, product-service, order-service,              │
│    orders, user-service                                        │
│         │                                                       │
│         │ feeds data into                                       │
│         ▼                                                       │
│  GRAFANA (port 3007)                                           │
│  Displays pretty dashboards & charts from Prometheus data.     │
│  Login: admin / admin                                          │
└─────────────────────────────────────────────────────────────────┘
```

---

## How a Typical Request Flows

**Example: A user logs in**

1. User types email + password in the browser at `localhost:3000`
2. The **Frontend** sends a `POST /api/auth/login` request
3. The **API Gateway** (port 3001) receives it and sees `/api/auth` → forwards to **Auth Service** (port 3002)
4. **Auth Service** checks the credentials against the `auth_db` database in **PostgreSQL**
5. If correct, Auth returns a **JWT token** (a signed digital pass that proves identity)
6. The token flows back through Gateway → Frontend → stored in the browser

**Example: A user places an order**

1. User clicks "Buy" in the browser
2. Frontend sends `POST /api/orders` → Gateway → **Order Service** (port 3004)
3. Order Service calls **Auth Service** to verify the JWT token is valid
4. Order Service calls **Product Service** to check the item is in stock
5. If both checks pass, it saves the order to `orders_db` in PostgreSQL
6. Success response flows back to the browser

---

## Networking

All containers live on an internal Docker network called `boutique-network`. They can talk to each other using their service names (like `http://auth:3002`). From outside Docker (your browser), you can only reach them via the mapped ports.

```
Your computer                 Docker internal network (boutique-network)
─────────────────            ─────────────────────────────────────────
localhost:3000  ──────────►  frontend (nginx)
localhost:3001  ──────────►  gateway
localhost:3002  ──────────►  auth
localhost:3003  ──────────►  product-service
localhost:3004  ──────────►  order-service
localhost:3005  ──────────►  orders
localhost:3006  ──────────►  user-service
localhost:3007  ──────────►  grafana
localhost:9090  ──────────►  prometheus
localhost:5432  ──────────►  postgres
```

---

## Data That Persists

Even if you stop and restart the containers, data is not lost. Docker **volumes** save the data to your computer's disk:

| Volume | What It Saves |
|---|---|
| `postgres_data` | All database tables and rows |
| `prometheus_data` | Historical metrics/performance data |
| `grafana_data` | Your dashboard configurations |

---

## Technology Stack

| Layer | Technology |
|---|---|
| Frontend | React (TypeScript), served by nginx |
| Backend services | Node.js with TypeScript |
| Database | PostgreSQL 15 |
| Metrics collection | Prometheus |
| Metrics visualization | Grafana |
| Containerization | Docker + Docker Compose |

---

## Quick Reference: Where to Open Things

| What | URL |
|---|---|
| The store website | http://localhost:3000 |
| API Gateway (direct) | http://localhost:3001 |
| Grafana dashboards | http://localhost:3007 (admin/admin) |
| Prometheus metrics | http://localhost:9090 |
