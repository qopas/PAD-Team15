# FAF Cab Management Platform

## 1. Service Boundaries
### Service Boundaries

Each microservice owns its data and API surface, is independently deployable and scalable, and has a single clear responsibility.

---

### User Management Service
- **Owns:**  
  User profiles, roles, FAF community membership mapping (name, nickname, group, role, Discord IDs).
- **Responsibility:**  
  Register users, store auth identifiers, provide identity metadata to other services.

---

### Notification Service
- **Owns:**  
  Notification delivery logic (email, Discord messages, in-app push).
- **Responsibility:**  
  Queueing, routing, retry logic, templates, targeted delivery, and audit logs.

---

### Tea Management Service
- **Owns:**  
  Consumables inventory (tea types, sugar, cups, markers), per-user consumption logs, stock thresholds, restock history.
- **Responsibility:**  
  Decrement inventory, generate low-stock alerts, per-user consumption tracking.

---

### Communication Service
- **Owns:**  
  Chat channels, direct messages, user nickname lookups, profanity/banned-word lists, ban enforcement, infractions log.
- **Responsibility:**  
  Message storage (short-term), moderation, event notifications for infractions.

---

### Cab Booking Service
- **Owns:**  
  Booking schedules for rooms (main room, kitchen), rules/limits, integration with Google Calendar.
- **Responsibility:**  
  Conflict detection, recurring bookings, calendar sync.

---

### Check-in Service
- **Owns:**  
  Door entry logs (CCTV facial-recog events), presence timeline, guest registration, unknown-person alerts.
- **Responsibility:**  
  Produce presence state and answer the critical question: *“Who has the key?”*

---

### Lost & Found Service
- **Owns:**  
  Lost/found posts, threaded comments, resolution flags.
- **Responsibility:**  
  Search, comment, and mark items as resolved.

---

### Fund Raising Service
- **Owns:**  
  Campaigns (goal, deadline, description), donation records.
- **Responsibility:**  
  Campaign lifecycle management, enforce deadlines, initiate purchase flows to other services.

---

### Budgeting Service
- **Owns:**  
  Treasury ledger (donations, purchases, expenses), debt book, report generation (CSV).
- **Responsibility:**  
  Maintain authoritative financial state and audits.

---

### Sharing Service
- **Owns:**  
  Inventory of shareable items (games, cords, kettles), rental/state tracking, ownership metadata, return/condition notes.
- **Responsibility:**  
  Manage lending lifecycle, track condition, create debt entries for damaged items.

---

## 2. Technologies & Communication Patterns

### Technologies & Communication Patterns

We selected **Java (Spring Boot)**, **Python (FastAPI / Django)**, **PHP**, and **.NET (ASP.NET Core)** to ensure diversity in tooling while balancing performance, developer productivity, and ecosystem maturity.  
Each microservice owns its database and communicates synchronously (REST / gRPC) or asynchronously (event broker).

---

### User Management Service
- **Technology:** PHP  
- **Database:** PostgreSQL  
- **Communication:** REST APIs (synchronous), publishes user lifecycle events to the message broker  
- **Motivation:**  
  PHP frameworks (e.g., Laravel) provide mature tooling for authentication and role-based access (student, teacher, admin).  
  PostgreSQL ensures relational integrity and consistency for user and role data.  
  PHP integrates well with third-party APIs for extended functionality.  
- **Trade-off:** PHP may have higher runtime overhead compared to compiled languages, but its ecosystem and rapid development speed outweigh this for identity management.

---

### Notification Service
- **Technology:** PHP  
- **Database:** Redis (for queuing) + SQL Server (for logs)  
- **Communication:** Consumes events from broker, pushes real-time notifications via WebSockets/SignalR, fallback via REST  
- **Motivation:**  
  PHP works well with Redis for fast message queuing and retry logic.  
  SQL Server is used for persistent logging and notification history.  
  This setup enables reliable real-time delivery with durable fallback storage.  
- **Trade-off:** Requires more infrastructure (Redis + SQL Server), but guarantees reliable and scalable push delivery.

---

### Tea Management Service
- **Technology:** .NET (ASP.NET Core)  
- **Database:** MongoDB  
- **Communication:** REST API for CRUD, emits `stock.low` events to broker  
- **Motivation:**  
  ASP.NET Core provides robustness, high performance, and enterprise-level reliability for inventory tracking.  
  MongoDB’s flexible schema allows easy handling of varying tea inventory attributes (e.g., packaging, origin, type).  
- **Trade-off:** Slightly higher development overhead compared to lightweight scripting languages, but long-term maintainability and performance are improved.

---

### Communication Service
- **Technology:** .NET (ASP.NET Core, SignalR)  
- **Database:** MongoDB (chat storage, short-lived history)  
- **Communication:** Real-time via WebSockets, REST for moderation endpoints, event publishing for infractions  
- **Motivation:**  
  ASP.NET Core + SignalR is ideal for scalable, low-latency chat.  
  MongoDB handles chat messages and channels flexibly, supporting variable message structures.  
- **Trade-off:** Requires tuning to avoid scaling bottlenecks, but real-time infra is native in .NET.

---

### Cab Booking Service
- **Technology:** Java  
- **Database:** PostgreSQL  
- **Communication:** REST API for booking management, emits `booking.created` events  
- **Motivation:**  
  Java + Spring Boot provides strong concurrency handling for booking operations.  
  PostgreSQL ensures ACID compliance, essential for transactions with time constraints.  
- **Trade-off:** Setup is more verbose compared to lightweight frameworks, but Java’s reliability and scalability are ideal for booking workloads.

---

### Check-in Service
- **Technology:** Java  
- **Database:** PostgreSQL  
- **Communication:** REST API + Event-driven (publishes `presence.enter/exit`), consumes from CCTV stream adapter  
- **Motivation:**  
  Java is well-suited for integrating with high-frequency event ingestion systems.  
  PostgreSQL ensures structured storage of entry/exit logs with consistent timestamps.  
- **Trade-off:** Requires careful optimization of concurrent requests but scales horizontally with ease.

---

### Lost & Found Service
- **Technology:** Python (Django REST Framework)  
- **Database:** MongoDB  
- **Communication:** REST API (CRUD endpoints), emits `post.resolved` events  
- **Motivation:**  
  Django REST accelerates building CRUD features and threaded discussions.  
  MongoDB supports flexible post structures, comments, and attachments without rigid schema requirements.  
- **Trade-off:** Slightly heavier runtime, but Django’s ORM and admin panel accelerate feature delivery.

---

### Budgeting Service
- **Technology:** Python  
- **Database:** PostgreSQL  
- **Communication:** REST API for querying/exporting, subscribes to donation and expense events  
- **Motivation:**  
  Python frameworks (e.g., Flask/Django) provide straightforward API development.  
  PostgreSQL ensures transactional integrity for budgets, expenses, and forecasting.  
  Built-in Python libraries simplify financial reporting and data exports.  
- **Trade-off:** Python’s performance is lower than compiled languages, but rapid iteration and ecosystem support make it effective for financial tracking.

---

### Fund Raising Service
- **Technology:** Java  
- **Database:** PostgreSQL  
- **Communication:** REST API, emits `fund.donation` and `fund.completed` events  
- **Motivation:**  
  Java + Spring Boot offers strong support for payment workflows and transaction management.  
  PostgreSQL ensures reliable storage of donations with ACID compliance.  
- **Trade-off:** Java setup is more verbose, but transaction safety and scalability are essential for fundraising.

---

### Sharing Service
- **Technology:** Java  
- **Database:** MongoDB  
- **Communication:** REST API, emits `sharing.rented` / `sharing.damaged` events, subscribes to budget service for debts  
- **Motivation:**  
  Java provides strong validation and concurrency control for sharing/rental operations.  
  MongoDB allows flexible object representation for items with varying properties (e.g., condition, size, availability).  
- **Trade-off:** Requires schema discipline to prevent inconsistencies, but flexibility supports evolving item types.


---


## Data Management Strategy

### Database Architecture
Each microservice maintains its own dedicated database to ensure data independence and service autonomy:

- **User Management Service**: PostgreSQL (relational data for user profiles and roles)
- **Tea Management Service**: MongoDB (flexible schema for inventory tracking)
- **Cab Booking Service**: PostgreSQL (structured booking data with time constraints)
- **Lost & Found Service**: MongoDB (flexible post structure with comments)
- **Fund Raising Service**: PostgreSQL (financial tracking requires ACID compliance)
- **Notification Service**: Redis (fast message queuing and temporary storage)
- **Communication Service**: MongoDB (chat messages and user interactions)
- **Check-in Service**: PostgreSQL (structured entry/exit logs)
- **Budgeting Service**: PostgreSQL (financial data with strict consistency)
- **Sharing Service**: MongoDB (flexible object tracking with varying properties)

### Communication Patterns
- **Synchronous Communication**: REST APIs for real-time operations
- **Asynchronous Communication**: HTTP webhooks and polling for notifications and event-driven updates
- **Service Discovery**: Consul for dynamic service registration and discovery
- **API Gateway**: Kong for request routing, authentication, and rate limiting

## Standardized Response Format

All services follow a consistent response format for both success and error cases:

### Success Response Format
```json
{
  "success": true,
  "message": "Operation completed successfully",
  "data": {
    // Response data object (optional, can be null or omitted for operations like DELETE)
  }
}
```

### Error Response Format
```json
{
  "success": false,
  "message": "Error description",
  "data": {
    "errorCode": "ERROR_CODE",
    "timestamp": "2025-01-15T10:30:00Z",
    "details": {}
  }
}
```

## API Endpoints Specification

### 1. User Management Service
**Base URL**: `/api/v1/users`

#### Endpoints

**GET /api/v1/users/{userId}**
```json
// Response (200 OK)
{
  "success": true,
  "message": "User retrieved successfully",
  "data": {
    "id": "string",
    "name": "string",
    "nickname": "string",
    "group": "string",
    "role": "student|teacher|admin",
    "discordId": "string",
    "email": "string",
    "createdAt": "2025-01-15T10:30:00Z",
    "updatedAt": "2025-01-15T10:30:00Z"
  }
}

// Error Response (404)
{
  "success": false,
  "message": "User not found",
  "data": {
    "errorCode": "USER_NOT_FOUND",
    "timestamp": "2025-01-15T10:30:00Z"
  }
}
```

**POST /api/v1/users**
```json
// Request Body
{
  "name": "string",
  "nickname": "string",
  "group": "string",
  "role": "student|teacher|admin",
  "discordId": "string",
  "email": "string"
}

// Response (201 Created)
{
  "success": true,
  "message": "User created successfully",
  "data": {
    "id": "string"
  }
}
```

**PUT /api/v1/users/{userId}**
```json
// Request Body
{
  "name": "string",
  "nickname": "string",
  "group": "string",
  "role": "student|teacher|admin"
}

// Response (200 OK)
{
  "success": true,
  "message": "User updated successfully",
  "data": null
}
```

**GET /api/v1/users/search**
```json
// Query Parameters: ?nickname=string&role=string

// Response (200 OK)
{
  "success": true,
  "message": "Users retrieved successfully",
  "data": {
    "users": [
      {
        "id": "string",
        "name": "string",
        "nickname": "string",
        "role": "string"
      }
    ],
    "total": "number"
  }
}
```

### 2. Tea Management Service
**Base URL**: `/api/v1/inventory`

**GET /api/v1/inventory/items**
```json
// Response (200 OK)
{
  "success": true,
  "message": "Inventory items retrieved successfully",
  "data": {
    "items": [
      {
        "id": "string",
        "name": "string",
        "category": "tea|sugar|cups|paper|markers",
        "quantity": "number",
        "unit": "string",
        "lowStockThreshold": "number",
        "lastRestocked": "2025-01-15T10:30:00Z"
      }
    ]
  }
}
```

**POST /api/v1/inventory/consume**
```json
// Request Body
{
  "userId": "string",
  "itemId": "string",
  "quantity": "number",
  "consumedAt": "2025-01-15T10:30:00Z"
}

// Response (200 OK)
{
  "success": true,
  "message": "Consumption recorded successfully",
  "data": {
    "remainingQuantity": "number",
    "lowStockAlert": "boolean"
  }
}
```

**GET /api/v1/inventory/usage/{userId}**
```json
// Response (200 OK)
{
  "success": true,
  "message": "Usage history retrieved successfully",
  "data": {
    "userId": "string",
    "usageHistory": [
      {
        "itemId": "string",
        "itemName": "string",
        "quantity": "number",
        "consumedAt": "2025-01-15T10:30:00Z"
      }
    ],
    "totalUsageThisMonth": "number"
  }
}
```

### 3. Cab Booking Service
**Base URL**: `/api/v1/bookings`

**POST /api/v1/bookings**
```json
// Request Body
{
  "userId": "string",
  "room": "main|kitchen",
  "startTime": "2025-01-15T10:30:00Z",
  "endTime": "2025-01-15T12:30:00Z",
  "purpose": "string",
  "attendees": ["string"],
  "googleCalendarEventId": "string"
}

// Response (201 Created)
{
  "success": true,
  "message": "Booking created successfully",
  "data": {
    "bookingId": "string",
    "status": "confirmed",
    "googleCalendarLink": "string"
  }
}
```

**GET /api/v1/bookings**
```json
// Query Parameters: ?date=2025-01-15&room=main

// Response (200 OK)
{
  "success": true,
  "message": "Bookings retrieved successfully",
  "data": {
    "bookings": [
      {
        "id": "string",
        "userId": "string",
        "userName": "string",
        "room": "string",
        "startTime": "2025-01-15T10:30:00Z",
        "endTime": "2025-01-15T12:30:00Z",
        "purpose": "string",
        "status": "confirmed|cancelled"
      }
    ]
  }
}
```

**DELETE /api/v1/bookings/{bookingId}**
```json
// Response (200 OK)
{
  "success": true,
  "message": "Booking cancelled successfully",
  "data": null
}
```

### 4. Lost & Found Service
**Base URL**: `/api/v1/lost-found`

**POST /api/v1/lost-found/posts**
```json
// Request Body
{
  "userId": "string",
  "type": "lost|found",
  "title": "string",
  "description": "string",
  "category": "electronics|clothing|books|other",
  "location": "string",
  "contactInfo": "string"
}

// Response (201 Created)
{
  "success": true,
  "message": "Post created successfully",
  "data": {
    "postId": "string"
  }
}
```

**GET /api/v1/lost-found/posts**
```json
// Query Parameters: ?type=lost&category=electronics&status=open

// Response (200 OK)
{
  "success": true,
  "message": "Posts retrieved successfully",
  "data": {
    "posts": [
      {
        "id": "string",
        "userId": "string",
        "userName": "string",
        "type": "lost|found",
        "title": "string",
        "description": "string",
        "category": "string",
        "location": "string",
        "status": "open|resolved",
        "createdAt": "2025-01-15T10:30:00Z",
        "commentsCount": "number"
      }
    ]
  }
}
```

**POST /api/v1/lost-found/posts/{postId}/comments**
```json
// Request Body
{
  "userId": "string",
  "content": "string"
}

// Response (201 Created)
{
  "success": true,
  "message": "Comment added successfully",
  "data": {
    "commentId": "string"
  }
}
```

### 5. Fund Raising Service
**Base URL**: `/api/v1/fundraising`

**POST /api/v1/fundraising/campaigns**
```json
// Request Body
{
  "createdBy": "string",
  "title": "string",
  "description": "string",
  "targetAmount": "number",
  "currency": "MDL",
  "deadline": "2025-02-15T23:59:59Z",
  "category": "equipment|supplies|events"
}

// Response (201 Created)
{
  "success": true,
  "message": "Campaign created successfully",
  "data": {
    "campaignId": "string"
  }
}
```

**POST /api/v1/fundraising/campaigns/{campaignId}/donate**
```json
// Request Body
{
  "userId": "string",
  "amount": "number",
  "message": "string"
}

// Response (200 OK)
{
  "success": true,
  "message": "Donation recorded successfully",
  "data": {
    "donationId": "string",
    "newTotal": "number",
    "percentageReached": "number"
  }
}
```

**GET /api/v1/fundraising/campaigns/{campaignId}**
```json
// Response (200 OK)
{
  "success": true,
  "message": "Campaign retrieved successfully",
  "data": {
    "id": "string",
    "title": "string",
    "description": "string",
    "targetAmount": "number",
    "currentAmount": "number",
    "percentageReached": "number",
    "status": "active|completed|expired",
    "deadline": "2025-02-15T23:59:59Z",
    "donationsCount": "number",
    "topDonors": [
      {
        "userId": "string",
        "userName": "string",
        "amount": "number"
      }
    ]
  }
}
```

### 6. Notification Service
**Base URL**: `/api/v1/notifications`

**POST /api/v1/notifications/send**
```json
// Request Body
{
  "recipientId": "string",
  "type": "low_stock|booking_reminder|chat_mention|violation|system",
  "title": "string",
  "message": "string",
  "priority": "low|medium|high",
  "actionRequired": "boolean",
  "metadata": {
    "serviceOrigin": "string",
    "relatedEntityId": "string"
  }
}

// Response (200 OK)
{
  "success": true,
  "message": "Notification sent successfully",
  "data": {
    "notificationId": "string",
    "status": "sent|queued|failed"
  }
}
```

**GET /api/v1/notifications/{userId}**
```json
// Query Parameters: ?unread=true&limit=50

// Response (200 OK)
{
  "success": true,
  "message": "Notifications retrieved successfully",
  "data": {
    "notifications": [
      {
        "id": "string",
        "type": "string",
        "title": "string",
        "message": "string",
        "priority": "string",
        "read": "boolean",
        "createdAt": "2025-01-15T10:30:00Z",
        "actionRequired": "boolean"
      }
    ],
    "unreadCount": "number"
  }
}
```

### 7. Communication Service
**Base URL**: `/api/v1/communication`

**POST /api/v1/communication/chats**
```json
// Request Body
{
  "createdBy": "string",
  "name": "string",
  "type": "private|group",
  "participants": ["string"],
  "isPublic": "boolean"
}

// Response (201 Created)
{
  "success": true,
  "message": "Chat created successfully",
  "data": {
    "chatId": "string",
    "inviteCode": "string"
  }
}
```

**POST /api/v1/communication/chats/{chatId}/messages**
```json
// Request Body
{
  "userId": "string",
  "content": "string",
  "messageType": "text|image|file"
}

// Response (201 Created)
{
  "success": true,
  "message": "Message sent successfully",
  "data": {
    "messageId": "string",
    "censored": "boolean",
    "violationDetected": "boolean",
    "processedContent": "string"
  }
}
```

**GET /api/v1/communication/chats/{chatId}/messages**
```json
// Query Parameters: ?limit=50&before=messageId

// Response (200 OK)
{
  "success": true,
  "message": "Messages retrieved successfully",
  "data": {
    "messages": [
      {
        "id": "string",
        "userId": "string",
        "userName": "string",
        "content": "string",
        "messageType": "string",
        "timestamp": "2025-01-15T10:30:00Z",
        "edited": "boolean"
      }
    ],
    "hasMore": "boolean"
  }
}
```

### 8. Check-in Service
**Base URL**: `/api/v1/checkin`

**POST /api/v1/checkin/entry**
```json
// Request Body
{
  "userId": "string",
  "entryType": "entry|exit",
  "method": "facial_recognition|manual|key_card",
  "timestamp": "2025-01-15T10:30:00Z"
}

// Response (200 OK)
{
  "success": true,
  "message": "Entry logged successfully",
  "data": {
    "logId": "string",
    "currentStatus": "inside|outside",
    "hasKey": "boolean"
  }
}
```

**POST /api/v1/checkin/guests**
```json
// Request Body
{
  "registeredBy": "string",
  "guestName": "string",
  "guestContact": "string",
  "visitPurpose": "string",
  "expectedDuration": "number"
}

// Response (201 Created)
{
  "success": true,
  "message": "Guest registered successfully",
  "data": {
    "guestId": "string",
    "accessCode": "string",
    "validUntil": "2025-01-15T18:30:00Z"
  }
}
```

**GET /api/v1/checkin/status**
```json
// Response (200 OK)
{
  "success": true,
  "message": "Status retrieved successfully",
  "data": {
    "currentOccupants": [
      {
        "userId": "string",
        "userName": "string",
        "entryTime": "2025-01-15T10:30:00Z",
        "hasKey": "boolean"
      }
    ],
    "keyHolder": {
      "userId": "string",
      "userName": "string"
    },
    "totalInside": "number"
  }
}
```

### 9. Budgeting Service
**Base URL**: `/api/v1/budget`

**GET /api/v1/budget/balance**
```json
// Response (200 OK)
{
  "success": true,
  "message": "Balance retrieved successfully",
  "data": {
    "currentBalance": "number",
    "currency": "MDL",
    "lastUpdated": "2025-01-15T10:30:00Z",
    "totalIncome": "number",
    "totalExpenses": "number",
    "monthlyChange": "number"
  }
}
```

**POST /api/v1/budget/transactions**
```json
// Request Body
{
  "type": "income|expense",
  "amount": "number",
  "category": "donations|purchases|fundraising|penalties",
  "description": "string",
  "source": "faf_ngo|partners|students",
  "relatedUserId": "string",
  "receiptUrl": "string"
}

// Response (201 Created)
{
  "success": true,
  "message": "Transaction recorded successfully",
  "data": {
    "transactionId": "string",
    "newBalance": "number"
  }
}
```

**GET /api/v1/budget/debt-book**
```json
// Response (200 OK)
{
  "success": true,
  "message": "Debt book retrieved successfully",
  "data": {
    "debts": [
      {
        "userId": "string",
        "userName": "string",
        "totalDebt": "number",
        "items": [
          {
            "description": "string",
            "amount": "number",
            "date": "2025-01-15T10:30:00Z",
            "resolved": "boolean"
          }
        ]
      }
    ]
  }
}
```

**GET /api/v1/budget/reports/csv**
```json
// Query Parameters: ?from=2025-01-01&to=2025-01-31&type=full

// Response (200 OK)
// Content-Type: text/csv
// Content-Disposition: attachment; filename="budget_report_2025_01.csv"
{
  "success": true,
  "message": "Report generated successfully",
  "data": {
    "downloadUrl": "string",
    "filename": "budget_report_2025_01.csv",
    "size": "number"
  }
}
```

### 10. Sharing Service
**Base URL**: `/api/v1/sharing`

**POST /api/v1/sharing/items**
```json
// Request Body
{
  "name": "string",
  "category": "games|electronics|kitchen|tools",
  "description": "string",
  "ownerId": "string",
  "ownerType": "user|cabinet",
  "condition": "excellent|good|fair|poor",
  "availableForRent": "boolean",
  "maxRentDuration": "number"
}

// Response (201 Created)
{
  "success": true,
  "message": "Item registered successfully",
  "data": {
    "itemId": "string"
  }
}
```

**POST /api/v1/sharing/items/{itemId}/rent**
```json
// Request Body
{
  "userId": "string",
  "requestedDuration": "number",
  "purpose": "string"
}

// Response (200 OK)
{
  "success": true,
  "message": "Item rented successfully",
  "data": {
    "rentalId": "string",
    "dueDate": "2025-01-20T18:00:00Z",
    "approved": "boolean"
  }
}
```

**POST /api/v1/sharing/items/{itemId}/return**
```json
// Request Body
{
  "rentalId": "string",
  "condition": "excellent|good|fair|poor|damaged",
  "notes": "string",
  "returnedAt": "2025-01-18T14:30:00Z"
}

// Response (200 OK)
{
  "success": true,
  "message": "Item returned successfully",
  "data": {
    "damageCharge": "number"
  }
}
```

**GET /api/v1/sharing/items**
```json
// Query Parameters: ?available=true&category=games&ownerId=string

// Response (200 OK)
{
  "success": true,
  "message": "Items retrieved successfully",
  "data": {
    "items": [
      {
        "id": "string",
        "name": "string",
        "category": "string",
        "description": "string",
        "ownerName": "string",
        "condition": "string",
        "available": "boolean",
        "currentRenter": "string",
        "dueDate": "2025-01-20T18:00:00Z"
      }
    ]
  }
}
```

## Inter-Service Communication Events

### Event-Driven Communication (Asynchronous)

**Event: UserCreated**
```json
{
  "eventType": "UserCreated",
  "timestamp": "2025-01-15T10:30:00Z",
  "data": {
    "userId": "string",
    "name": "string",
    "nickname": "string",
    "role": "string"
  }
}
```
*Consumed by*: Communication Service, Notification Service

**Event: InventoryLowStock**
```json
{
  "eventType": "InventoryLowStock",
  "timestamp": "2025-01-15T10:30:00Z",
  "data": {
    "itemId": "string",
    "itemName": "string",
    "currentQuantity": "number",
    "threshold": "number"
  }
}
```
*Consumed by*: Notification Service, Budgeting Service

**Event: BookingCreated**
```json
{
  "eventType": "BookingCreated",
  "timestamp": "2025-01-15T10:30:00Z",
  "data": {
    "bookingId": "string",
    "userId": "string",
    "room": "string",
    "startTime": "2025-01-15T10:30:00Z",
    "endTime": "2025-01-15T12:30:00Z"
  }
}
```
*Consumed by*: Notification Service, Check-in Service

**Event: ViolationDetected**
```json
{
  "eventType": "ViolationDetected",
  "timestamp": "2025-01-15T10:30:00Z",
  "data": {
    "userId": "string",
    "violationType": "excessive_usage|inappropriate_content|property_damage",
    "severity": "minor|major|severe",
    "details": "string"
  }
}
```
*Consumed by*: Notification Service, Budgeting Service (for penalties)

**Event: FundraisingCompleted**
```json
{
  "eventType": "FundraisingCompleted",
  "timestamp": "2025-01-15T10:30:00Z",
  "data": {
    "campaignId": "string",
    "totalRaised": "number",
    "targetAmount": "number",
    "itemDescription": "string",
    "leftoverFunds": "number"
  }
}
```
*Consumed by*: Budgeting Service, Sharing Service, Notification Service

**Event: DebtIncurred**
```json
{
  "eventType": "DebtIncurred",
  "timestamp": "2025-01-15T10:30:00Z",
  "data": {
    "userId": "string",
    "amount": "number",
    "reason": "string",
    "itemId": "string"
  }
}
```
*Consumed by*: Budgeting Service, Notification Service

## Authentication & Authorization

All services implement JWT-based authentication with the following token structure:

```json
{
  "userId": "string",
  "role": "student|teacher|admin",
  "nickname": "string",
  "permissions": ["string"],
  "iat": "number",
  "exp": "number"
}
```

### Authorization Levels
- **Student**: Basic access to most services, limited administrative functions
- **Teacher**: Enhanced access including booking priority and extended usage limits
- **Admin**: Full access to all services including financial data and user management

## Rate Limiting

- **Standard endpoints**: 100 requests per minute per user
- **File uploads**: 10 requests per minute per user
- **Notifications**: 50 requests per minute per user
- **Search endpoints**: 200 requests per minute per user

## Data Validation

All services implement comprehensive input validation using JSON Schema or equivalent validation frameworks specific to their technology stack. Common validation rules include:

- **User IDs**: UUID format validation
- **Timestamps**: ISO 8601 format validation
- **Amounts**: Positive numbers with appropriate decimal precision
- **Enum fields**: Strict validation against predefined values
- **Required fields**: Non-null validation for mandatory fields

## HTTP Status Codes

All services use standard HTTP status codes consistently:

- **200 OK**: Successful GET, PUT, PATCH operations
- **201 Created**: Successful POST operations creating new resources
- **204 No Content**: Successful DELETE operations
- **400 Bad Request**: Invalid request data or validation errors
- **401 Unauthorized**: Missing or invalid authentication
- **403 Forbidden**: Insufficient permissions
- **404 Not Found**: Resource not found
- **409 Conflict**: Resource conflicts (e.g., duplicate booking times)
- **422 Unprocessable Entity**: Valid request format but business logic errors
- **500 Internal Server Error**: Server-side errors

## Service Health Monitoring

Each service exposes health check endpoints:

**GET /health**
```json
{
  "success": true,
  "message": "Service is healthy",
  "data": {
    "status": "healthy",
    "timestamp": "2025-01-15T10:30:00Z",
    "version": "1.0.0",
    "dependencies": {
      "database": "healthy",
      "externalServices": "healthy"
    }
  }
}
```


# 📘 Project Workflow & Contribution Guidelines

This project follows a standardized CPR (Collaborative Pull Request) workflow to ensure clean code, traceability, and high-quality delivery.

---

## 🚀 GitHub Workflow Setup (CPR)

### 🔧 Repository Configuration

- Main Branches:  
  - main: Production-ready code  
  - development: Integration and staging branch

- Pull Request Rules:
  - A minimum of 1 approvals is required before merging any pull request.

- Merge Strategy:  
  - We use Merge commits (GitHub’s default strategy)  
  - This preserves full commit history.

---

## 🌱 Branch Naming Strategy

All feature branches must follow this naming convention:
feature/[surname]/[task-id]
Example:
feature/smith/PAD-123
Other branch types (optional):
bugfix/[surname]/[task-id]
hotfix/[surname]/[task-id]


## 🔀 Pull Request Guidelines

- PR Title:  
  Use the task ID as the title.  
  Example: PAD-123

- PR Description:  
  Include a short description of the change and a link to the related task.  
  Example: Adds user authentication middleware.
Related task: https://github/project/PAD-123
---

## ✅ Commit Convention

We follow [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/) to ensure consistent and parseable commit messages.

Format:
<type>(scope): <short description>
Examples:

feat(auth): add JWT login
fix(api): handle null user response
chore: update dependencies
Common types:  
feat, fix, docs, style, refactor, test, chore

---

## 🧪 Test Coverage

- All new features must include unit test