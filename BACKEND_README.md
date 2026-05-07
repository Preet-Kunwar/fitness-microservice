# Fitness Microservices Backend

A production-ready, cloud-native fitness tracking platform built with Spring Boot and Spring Cloud microservices. The system lets users log workouts, validates them against user records, and uses Google Gemini AI to generate personalized fitness recommendations — all secured behind an OAuth2/Keycloak gateway.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Services](#services)
- [Tech Stack](#tech-stack)
- [Prerequisites](#prerequisites)
- [Infrastructure Setup](#infrastructure-setup)
- [Configuration](#configuration)
- [Running the Services](#running-the-services)
- [API Reference](#api-reference)
- [Message Flow](#message-flow)
- [Security](#security)
- [Project Structure](#project-structure)

---

## Architecture Overview

```
Client (React Frontend)
        │
        ▼
  [API Gateway :8080]  ←── Keycloak OAuth2 JWT Validation
        │
        ├──────────────────────────────────────┐
        ▼                                      ▼
[User Service :8081]              [Activity Service :8082]
  PostgreSQL                          MongoDB
                                        │
                                        │ RabbitMQ (activity.queue)
                                        ▼
                                  [AI Service :8083]
                                    MongoDB + Gemini API

  [Eureka Server :8761]  ← Service Discovery (all services register here)
  [Config Server :8888]  ← Centralised configuration
```

All inter-service communication goes through Eureka-based load-balanced `WebClient` calls. The API Gateway performs JWT validation and automatically syncs first-time Keycloak users into the User Service.

---

## Services

### 1. Config Server (`configserver` — port 8888)
Central configuration server using Spring Cloud Config with `native` profile. Config files for every service are stored under `src/main/resources/config/`.

### 2. Eureka Server (`eureka` — port 8761)
Netflix Eureka service registry. All microservices register here on startup, enabling client-side load balancing.

### 3. API Gateway (`gateway` — port 8080)
Spring Cloud Gateway with:
- OAuth2 Resource Server (JWT validation against Keycloak)
- `KeycloakUserSyncFilter` — intercepts every request, decodes the JWT, and auto-registers new users in the User Service
- CORS configuration permitting `http://localhost:5173`
- Route definitions for all downstream services

### 4. User Service (`userservice` — port 8081)
Manages user profiles persisted in PostgreSQL.

| Endpoint | Method | Description |
|---|---|---|
| `/api/users/register` | POST | Register a new user |
| `/api/users/{userId}` | GET | Get user profile |
| `/api/users/{userId}/validate` | GET | Check if a Keycloak user exists |

### 5. Activity Service (`activityservice` — port 8082)
Handles workout activity CRUD backed by MongoDB.

| Endpoint | Method | Description |
|---|---|---|
| `POST /api/activities` | POST | Log a new activity |
| `GET /api/activities` | GET | List authenticated user's activities |
| `GET /api/activities/{id}` | GET | Get single activity |

On every successful activity save, the record is published to RabbitMQ for async AI processing.

### 6. AI Service (`aiservice` — port 8083)
Listens to the RabbitMQ `activity.queue`, sends activity data to Google Gemini, parses the structured JSON response, and saves recommendations to MongoDB.

| Endpoint | Method | Description |
|---|---|---|
| `GET /api/recommendations/user/{userId}` | GET | All recommendations for a user |
| `GET /api/recommendations/activity/{activityId}` | GET | Recommendation for a specific activity |

---

## Tech Stack

| Layer | Technology |
|---|---|
| Framework | Spring Boot 3.4.3 |
| Service Discovery | Spring Cloud Netflix Eureka |
| Config Management | Spring Cloud Config |
| API Gateway | Spring Cloud Gateway |
| Auth | Keycloak + Spring Security OAuth2 Resource Server |
| Async Messaging | RabbitMQ + Spring AMQP |
| AI Integration | Google Gemini API (via WebClient) |
| Databases | PostgreSQL (users), MongoDB (activities, recommendations) |
| Build Tool | Apache Maven 3.9.9 |
| Java Version | Java 17 (config/eureka), Java 23 (all other services) |
| Boilerplate Reduction | Lombok |

---

## Prerequisites

Make sure the following are installed and running before you start:

- **JDK 17+** (JDK 23 recommended)
- **Maven 3.9+** (or use the included `mvnw` wrapper)
- **PostgreSQL** — database `fitness_user_db`, user `postgres`, password `admin@123`
- **MongoDB** — running on `localhost:27017`
- **RabbitMQ** — running on `localhost:5672` (default guest/guest credentials)
- **Keycloak** — running on `localhost:8181` with a realm named `fitness-oauth2` and client `oauth2-pkce-client`
- **Google Gemini API key** — set as environment variables

---

## Infrastructure Setup

### PostgreSQL
```sql
CREATE DATABASE fitness_user_db;
```

### Keycloak
1. Start Keycloak: `./kc.sh start-dev --http-port=8181`
2. Create realm: `fitness-oauth2`
3. Create client: `oauth2-pkce-client`
   - Client authentication: OFF (public client)
   - Valid redirect URIs: `http://localhost:5173/*`
   - Web origins: `http://localhost:5173`
4. Create a test user with email/password credentials

### RabbitMQ
The default Docker setup works fine:
```bash
docker run -d --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:management
```

### MongoDB
```bash
docker run -d --name mongodb -p 27017:27017 mongo
```

---

## Configuration

All service-specific configuration lives in the Config Server at:
```
configserver/src/main/resources/config/
├── activity-service.yml
├── ai-service.yml
├── api-gateway.yml
└── user-service.yml
```

### Environment Variables for AI Service

The AI service requires two environment variables — do **not** hard-code keys:

```bash
export GEMINI_API_URL=https://generativelanguage.googleapis.com/v1beta/models/gemini-pro:generateContent?key=
export GEMINI_API_KEY=your_actual_api_key_here
```

These are referenced in `ai-service.yml`:
```yaml
gemini:
  api:
    url: ${GEMINI_API_URL}
    key: ${GEMINI_API_KEY}
```

---

## Running the Services

Services **must** be started in this order:

```bash
# 1. Config Server (all other services pull config from here)
cd configserver
./mvnw spring-boot:run

# 2. Eureka Server (service registry)
cd eureka
./mvnw spring-boot:run

# 3. User Service
cd userservice
./mvnw spring-boot:run

# 4. Activity Service
cd activityservice
./mvnw spring-boot:run

# 5. AI Service
cd aiservice
./mvnw spring-boot:run

# 6. API Gateway (last, after all downstream services are up)
cd gateway
./mvnw spring-boot:run
```

After startup, the Eureka dashboard is available at: `http://localhost:8761`

---

## API Reference

All requests go through the gateway at `http://localhost:8080`. Every request must carry a valid JWT:

```
Authorization: Bearer <access_token>
```

The gateway injects the `X-User-ID` header automatically from the JWT `sub` claim — you do not need to send it manually from the frontend.

### Activity Endpoints

**POST** `/api/activities`
```json
{
  "type": "RUNNING",
  "duration": 30,
  "caloriesBurned": 350,
  "startTime": "2025-05-07T08:00:00",
  "additionalMetrics": {
    "distance": 5.2,
    "avgHeartRate": 145
  }
}
```

Supported activity types: `RUNNING`, `WALKING`, `CYCLING`, `SWIMMING`, `WEIGHT_TRAINING`, `YOGA`, `HIIT`, `CARDIO`, `STRETCHING`, `OTHER`

**GET** `/api/activities` — returns all activities for the authenticated user

**GET** `/api/activities/{id}` — returns a single activity

### Recommendation Endpoints

**GET** `/api/recommendations/user/{userId}` — all AI recommendations for a user

**GET** `/api/recommendations/activity/{activityId}` — AI recommendation for a specific activity

Sample recommendation response:
```json
{
  "id": "...",
  "activityId": "...",
  "userId": "...",
  "activityType": "RUNNING",
  "recommendation": "Overall: Good steady-state run...\n\nPace: ...",
  "improvements": ["Endurance: Consider adding interval training"],
  "suggestions": ["Hill Repeats: Run uphill for 30s, recover, repeat 8x"],
  "safety": ["Warm up for 5-10 minutes", "Stay hydrated"],
  "createdAt": "2025-05-07T09:30:00"
}
```

---

## Message Flow

1. User POSTs an activity → Activity Service validates the user via User Service
2. Activity is saved to MongoDB
3. Activity Service publishes the saved `Activity` object to RabbitMQ exchange `fitness.exchange` with routing key `activity.tracking`
4. AI Service `ActivityMessageListener` receives the message from `activity.queue`
5. `ActivityAIService` constructs a detailed prompt and calls Gemini
6. The JSON response from Gemini is parsed into structured fields (analysis, improvements, suggestions, safety)
7. A `Recommendation` document is saved to MongoDB
8. Frontend can now fetch the recommendation via `/api/recommendations/activity/{activityId}`

If RabbitMQ is unavailable during step 3, the error is logged but the activity save is **not** rolled back — the system degrades gracefully.

---

## Security

- All gateway routes require a valid Keycloak JWT (`anyExchange().authenticated()`)
- JWT signature is verified against Keycloak's JWK endpoint: `http://localhost:8181/realms/fitness-oauth2/protocol/openid-connect/certs`
- The `KeycloakUserSyncFilter` runs on every authenticated request and idempotently creates the user in the User Service if they don't exist yet
- CSRF is disabled (stateless REST API)
- CORS is restricted to `http://localhost:5173` for `GET`, `POST`, `PUT`, `DELETE`, `OPTIONS` on `/api/**`
- Passwords stored in the User Service are placeholder values — actual authentication is fully delegated to Keycloak

---

## Project Structure

```
fitness-microservice/
├── configserver/          # Spring Cloud Config Server
│   └── src/main/resources/config/
│       ├── activity-service.yml
│       ├── ai-service.yml
│       ├── api-gateway.yml
│       └── user-service.yml
├── eureka/                # Eureka Service Registry
├── gateway/               # API Gateway + Security + User Sync
│   └── src/main/java/com/fitness/gateway/
│       ├── KeycloakUserSyncFilter.java
│       ├── SecurityConfig.java
│       └── user/           # WebClient to User Service
├── userservice/           # User management (PostgreSQL)
│   └── src/main/java/com/fitness/userservice/
│       ├── controller/
│       ├── dto/
│       ├── model/
│       ├── repository/
│       └── service/
├── activityservice/       # Activity CRUD + RabbitMQ publisher (MongoDB)
│   └── src/main/java/com/fitness/activityservice/
│       ├── config/         # RabbitMQ, MongoDB, WebClient configs
│       ├── controller/
│       ├── dto/
│       ├── model/
│       └── service/
└── aiservice/             # RabbitMQ consumer + Gemini + Recommendations (MongoDB)
    └── src/main/java/com/fitness/aiservice/
        ├── config/         # RabbitMQ, MongoDB configs
        ├── controller/
        ├── model/
        ├── repository/
        └── service/
            ├── ActivityAIService.java
            ├── ActivityMessageListener.java
            ├── GeminiService.java
            └── RecommendationService.java
```
