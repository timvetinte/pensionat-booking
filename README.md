# B2 Pensionat — Booking Service

The **booking service** is the main entry point of the B2 Pensionat system, a small
guesthouse (*pensionat*) management platform built as three cooperating Spring Boot
microservices. It exposes the room, booking, customer and review functionality as a
single REST API (and a minimal static web UI) by orchestrating calls to the two
supporting services.

This document describes the system as a whole: what each service is responsible for,
how they talk to each other, and how to start all three together with Docker Compose
or Kubernetes. For service-specific details, see the smaller READMEs shipped with
[b2_pensionat_customer](../b2_pensionat_customer) and [b2_pensionat_reviews](../b2_pensionat_reviews).

## System overview

| Service | Port | Responsibility | Database |
|---|---|---|---|
| **booking** (this service) | `8080` | Rooms, bookings, and the public API/UI. Calls out to `customer` and `reviews` for anything it doesn't own. | `pensionat_booking` (MySQL) |
| [customer](../b2_pensionat_customer) | `8081` | Customer records (CRUD). | `pensionat_customer` (MySQL) |
| [reviews](../b2_pensionat_reviews) | `8082` | Room reviews (create/list). | `pensionat_reviews` (MySQL) |

Booking is the only service meant to be reached from outside the cluster/network;
`customer` and `reviews` are internal microservices that only booking calls.

```
                 ┌────────────────────┐
   Browser ────► │   booking-service   │ :8080
                 │  (rooms, bookings,  │
                 │   REST API + UI)    │
                 └─────────┬───────────┘
                     RestTemplate (HTTP)
              ┌────────────┴────────────┐
              ▼                         ▼
   ┌─────────────────────┐   ┌─────────────────────┐
   │  customer-service    │   │  reviews-service     │
   │  :8081                │   │  :8082                │
   └──────────┬───────────┘   └──────────┬───────────┘
              ▼                          ▼
        booking-db*               reviews-db
        (MySQL)                   (MySQL)

   booking-service also has its own DB for rooms/bookings.
```

### How the services talk to each other

Booking never touches the customer or reviews databases directly — it calls their
HTTP APIs over plain REST using `RestTemplate`, and re-exposes (or wraps) the results
under its own `/api/*` and `/reviews` endpoints:

- `CustomerServiceClient` → calls `customer-service` at `${customer-service.base-url}`
  - `GET  /customers/all`
  - `GET  /customers/{id}`
  - `POST /customers/register`
  - `PUT  /customers/editCst`
  - `DELETE /customers/delete/{id}`
- `ReviewServiceClient` → calls `reviews-service` at `${reviews-service.base-url}`
  - `GET  /review`
  - `POST /review`

The base URLs are configured per environment (see [Configuration](#configuration))
so the same code works whether the services run on `localhost`, as Docker Compose
containers, or as Kubernetes Services.

Booking additionally guards against deleting a customer that still has bookings
(returns `409 Conflict`), so it does read booking state before delegating the delete
to the customer service.

## Booking service API

All endpoints are served under `http://localhost:8080` when run standalone.

### Rooms — `/api/rooms`

| Method | Path | Description |
|---|---|---|
| GET | `/api/rooms` | List all rooms. |

### Bookings — `/api/bookings`

| Method | Path | Description |
|---|---|---|
| GET | `/api/bookings` | List all bookings. |
| GET | `/api/bookings/available-rooms?startDate=&endDate=&doubleRoom=` | Check/list rooms available for a date range. |
| POST | `/api/bookings?startDate=&endDate=&isDoubleRoom=&customerId=&extraBeds=` | Create a booking. |
| PUT | `/api/bookings/{id}?startDate=&endDate=` | Change the dates of an existing booking. |
| DELETE | `/api/bookings/{bookingID}` | Cancel a booking. |

### Customers — `/api/customers` (proxied to the customer service)

| Method | Path | Description |
|---|---|---|
| GET | `/api/customers` | List all customers. |
| GET | `/api/customers/{id}` | Get a customer by id. |
| POST | `/api/customers/register` | Register a new customer. |
| PUT | `/api/customers/edit` | Edit a customer. |
| DELETE | `/api/customers/delete/{id}` | Delete a customer (blocked with `409` if they have active bookings). |

### Reviews — `/reviews` (proxied to the reviews service)

| Method | Path | Description |
|---|---|---|
| GET | `/reviews` | List all reviews. |
| POST | `/reviews` | Create a review. |

Interactive API docs (Swagger UI, via springdoc) are available at
`/swagger-ui.html` once the service is running.

### Web UI

A minimal static UI (Thymeleaf/HTML/JS under `src/main/resources/static`) is served
from `/` and lets you browse rooms, manage bookings, register/edit customers, and
leave reviews without calling the API directly.

## Tech stack

- Java 25, Spring Boot 4
- Spring Web MVC, Spring Data JPA, Spring Validation, Thymeleaf
- MySQL (one schema per service)
- springdoc-openapi (Swagger UI)
- Testcontainers + JUnit 5 for integration tests
- Docker / Docker Compose, Kubernetes manifests

## Configuration

Each service reads its datasource and peer URLs from `application.properties`
(local/dev defaults) and `application-prod.properties` (used inside containers via
`--spring.profiles.active=prod` / `SPRING_PROFILES_ACTIVE=prod`).

Relevant booking properties:

| Property | Purpose | Example (prod profile) |
|---|---|---|
| `server.port` | Port booking listens on | `8080` |
| `spring.datasource.url` | Booking's own MySQL connection | `jdbc:mysql://booking-db/pensionat_booking` |
| `customer-service.base-url` | Where to reach the customer service | `http://customer-service:8081` |
| `reviews-service.base-url` | Where to reach the reviews service | `http://reviews-service:8082` |

Credentials are supplied via the active Spring profile / environment and are not
committed with real values in this document — check each service's
`application-prod.properties` for the container defaults used by Compose/K8s below.

## Running the whole system

The system is three Spring Boot apps, each with its own MySQL database — six
containers/pods in total. Booking is the only one you need to expose publicly.

### Option A — Docker Compose

This repository's `docker-compose.yml` builds and wires up all three services and
their databases in one go. It expects the three project folders to be checked out
as siblings on disk:

```
school/
├── b2_pensionat_booking/     (contains docker-compose.yml)
├── b2_pensionat_customer/
└── b2_pensionat_reviews/
```

From `b2_pensionat_booking/`:

```bash
docker compose up --build
```

This starts:

- `booking-db`, `customer-db`, `reviews-db` — MySQL 8 instances, one per service
- `booking-service` — built from this repo, exposed on `localhost:8080`
- `customer-service` — built from `../b2_pensionat_customer`, exposed on `localhost:8081`
- `reviews-service` — built from `../b2_pensionat_reviews`, exposed on `localhost:8082`

All three apps run with `SPRING_PROFILES_ACTIVE=prod`, which points them at the
Compose service names (`booking-db`, `customer-service`, `reviews-service`, ...)
instead of `localhost`.

Once it's up, open **http://localhost:8080** for the UI, or **http://localhost:8080/swagger-ui.html**
for the API docs. Tear everything down with:

```bash
docker compose down -v
```

### Option B — Kubernetes

Each service ships plain Deployment + Service manifests under its own `k8s/`
folder. Booking's `k8s/` folder contains manifests for *all six* workloads
(the three services and their three databases), so applying it stands up the
entire system in one namespace:

```bash
# from b2_pensionat_booking/
kubectl apply -f k8s/
```

This creates:

- `booking-db`, `customer-db`, `reviews-db` Deployments + Services (MySQL, port `3306`)
- `booking-service` (port `8080`), `customer-service` (port `8081`),
  `reviews-service` (port `8082`) Deployments + Services

The service manifests use `imagePullPolicy: Never`, meaning the images
(`booking-service`, `customer-service`, `reviews-service`) must already exist in the
cluster's local image store before applying — e.g. build them and load them into
your cluster (`docker build` + `kind load docker-image ...`, or point your daemon at
the cluster's Docker/containerd runtime, such as `eval $(minikube docker-env)` before
building).

To reach booking from outside the cluster, port-forward it:

```bash
kubectl port-forward svc/booking-service 8080:8080
```

Then browse **http://localhost:8080**.

To remove everything:

```bash
kubectl delete -f k8s/
```

> `b2_pensionat_customer/k8s/` also contains a standalone copy of the
> customer-service + customer-db manifests, useful for running/testing that
> service on its own.

## Building and running booking standalone

```bash
./mvnw clean package
java -jar target/*.jar
```

By default (no profile) it expects a MySQL instance reachable per
`application.properties` and the customer service on `http://localhost:8081`.
Adjust `application.properties` or pass overrides via `-D` / environment variables
to point at your own database and peer services.

## Tests

```bash
./mvnw test
```

Integration tests spin up MySQL via Testcontainers, so Docker must be available
locally.
