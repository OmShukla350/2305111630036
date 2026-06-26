# Stage 1

# Notification System Design

## Overview

This document defines the REST APIs required for a campus notification platform. The platform allows authenticated students to receive notifications related to placements, results, and events.

### Features Supported

- Fetch notifications
- View a single notification
- Mark notifications as read
- Mark all notifications as read
- Get unread notification count
- Create notifications (Admin/Internal Service)
- Real-time notification delivery

---

## Base URL

```
/api/v1
```

---

## Authentication

All APIs are protected.

### Headers

```http
Authorization: Bearer <JWT_TOKEN>
Content-Type: application/json
Accept: application/json
```

---

# 1. Fetch Notifications

### Endpoint

```http
GET /api/v1/notifications
```

### Description

Returns paginated notifications for the authenticated student.

### Query Parameters

| Parameter | Type | Required | Description |
|------------|------|----------|-------------|
| page | Integer | No | Page number (Default = 1) |
| limit | Integer | No | Number of records per page (Default = 10) |
| notification_type | String | No | Filter by Event, Placement or Result |
| isRead | Boolean | No | Filter read/unread notifications |
| sort | String | No | Sort order (newest or oldest) |

### Request Body

None

### Success Response

```json
{
  "success": true,
  "page": 1,
  "limit": 10,
  "totalNotifications": 135,
  "totalPages": 14,
  "notifications": [
    {
      "id": "n101",
      "title": "Amazon Hiring",
      "message": "Amazon hiring for SDE-1",
      "type": "Placement",
      "isRead": false,
      "createdAt": "2026-04-22T17:51:18Z"
    }
  ]
}
```

### Status Codes

```
200 OK
400 Bad Request
401 Unauthorized
500 Internal Server Error
```

---

# 2. Get Notification Details

### Endpoint

```http
GET /api/v1/notifications/{notificationId}
```

### Description

Returns complete details of a specific notification.

### Success Response

```json
{
  "success": true,
  "notification": {
    "id": "n101",
    "title": "Amazon Hiring",
    "message": "Amazon hiring for SDE-1",
    "type": "Placement",
    "isRead": false,
    "createdAt": "2026-04-22T17:51:18Z"
  }
}
```

### Status Codes

```
200 OK
401 Unauthorized
404 Not Found
500 Internal Server Error
```

---

# 3. Mark Notification as Read

### Endpoint

```http
PATCH /api/v1/notifications/{notificationId}/read
```

### Description

Marks a notification as read.

### Request Body

None

### Success Response

```json
{
  "success": true,
  "message": "Notification marked as read."
}
```

### Status Codes

```
200 OK
401 Unauthorized
404 Not Found
500 Internal Server Error
```

---

# 4. Mark All Notifications as Read

### Endpoint

```http
PATCH /api/v1/notifications/read-all
```

### Description

Marks all unread notifications of the authenticated student as read.

### Success Response

```json
{
  "success": true,
  "message": "All notifications marked as read."
}
```

### Status Codes

```
200 OK
401 Unauthorized
500 Internal Server Error
```

---

# 5. Get Unread Notification Count

### Endpoint

```http
GET /api/v1/notifications/unread-count
```

### Description

Returns the total number of unread notifications.

### Success Response

```json
{
  "success": true,
  "unreadCount": 8
}
```

### Status Codes

```
200 OK
401 Unauthorized
500 Internal Server Error
```

---

# 6. Create Notification (Admin/Internal Service)

### Endpoint

```http
POST /api/v1/notifications
```

### Description

Creates a new notification. This API is intended for administrators or internal services.

### Request Body

```json
{
  "title": "Amazon Hiring",
  "message": "Amazon is hiring Software Engineers.",
  "type": "Placement",
  "recipientIds": [
    "stu101",
    "stu102",
    "stu103"
  ]
}
```

### Success Response

```json
{
  "success": true,
  "message": "Notification created successfully."
}
```

### Status Codes

```
201 Created
400 Bad Request
401 Unauthorized
403 Forbidden
500 Internal Server Error
```

---

# Error Response Format

All APIs follow a common error response format.

```json
{
  "success": false,
  "message": "Resource not found.",
  "errorCode": "NOT_FOUND"
}
```

---

# Real-Time Notification Design

The application uses **WebSockets** to deliver notifications instantly to connected students.

## Workflow

1. Student logs into the application.
2. A WebSocket connection is established.
3. When an administrator creates a notification, the backend:
   - Saves the notification in the database.
   - Publishes an event to connected clients.
4. Online users receive the notification immediately.
5. Offline users receive the notification when they reconnect by fetching unread notifications through the REST API.

### Advantages

- Instant notification delivery
- Eliminates frequent polling
- Reduces unnecessary API requests
- Improves user experience

### Fallback Strategy

If the WebSocket connection is unavailable, the frontend periodically polls:

```http
GET /api/v1/notifications?page=1&limit=10
```

to synchronize missed notifications.

---
## Logging Strategy

Each API operation will invoke the reusable logging middleware.

Examples

Notification Fetch

Log(
"backend",
"info",
"controller",
"Fetching notifications"
)

Notification Created

Log(
"backend",
"info",
"service",
"Notification created successfully"
)

Database Error

Log(
"backend",
"error",
"db",
"Database connection failed"
)

# API Design Principles

- RESTful endpoint naming conventions
- Resource-based URLs
- JWT authentication
- Pagination for large datasets
- Filtering support
- Standard HTTP status codes
- Consistent JSON response format
- Versioned APIs (`/api/v1`)
- Real-time communication using WebSockets
---
# Stage 2

## Database Selection

### Recommended Database: PostgreSQL

For the Campus Notification Platform, **PostgreSQL** is selected as the primary persistent storage because notifications are highly structured, require strong consistency, and involve frequent filtering, sorting, and updates.

### Reasons for Choosing PostgreSQL

* ACID-compliant transactions ensure reliable data storage.
* Excellent support for indexing, filtering, and sorting.
* Optimized for relational data and complex queries.
* Supports foreign key constraints to maintain data integrity.
* Provides JSON support for optional notification metadata.
* Scales well with indexing, partitioning, and read replicas.

---

# Database Schema

The system uses a normalized relational database with three tables.

## 1. Users Table

Stores student information.

| Column     | Type         | Constraints               |
| ---------- | ------------ | ------------------------- |
| user_id    | UUID         | Primary Key               |
| full_name  | VARCHAR(100) | NOT NULL                  |
| email      | VARCHAR(255) | UNIQUE                    |
| created_at | TIMESTAMP    | DEFAULT CURRENT_TIMESTAMP |

---

## 2. Notifications Table

Stores notification details shared across one or more students.

| Column            | Type                               | Constraints               |
| ----------------- | ---------------------------------- | ------------------------- |
| notification_id   | UUID                               | Primary Key               |
| title             | VARCHAR(255)                       | NOT NULL                  |
| message           | TEXT                               | NOT NULL                  |
| notification_type | ENUM('Placement','Result','Event') | NOT NULL                  |
| priority          | SMALLINT                           | DEFAULT 1                 |
| created_at        | TIMESTAMP                          | DEFAULT CURRENT_TIMESTAMP |

---

## 3. User_Notifications Table

Maps notifications to individual students while maintaining each student's read status.

| Column          | Type      | Constraints                                  |
| --------------- | --------- | -------------------------------------------- |
| id              | UUID      | Primary Key                                  |
| user_id         | UUID      | Foreign Key → Users(user_id)                 |
| notification_id | UUID      | Foreign Key → Notifications(notification_id) |
| is_read         | BOOLEAN   | DEFAULT FALSE                                |
| read_at         | TIMESTAMP | NULL                                         |
| delivered_at    | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP                    |

---

# Entity Relationship

```
Users
------
user_id (PK)

        1
        |
        |
        |
        N

User_Notifications
-------------------
id (PK)
user_id (FK)
notification_id (FK)
is_read
read_at

        N
        |
        |
        |
        1

Notifications
--------------
notification_id (PK)
title
message
notification_type
priority
created_at
```

---

# Why this Schema?

Instead of storing duplicate notification records for every student, a single notification is stored once in the `Notifications` table while the `User_Notifications` table maintains each student's read status.

This design:

* Reduces data duplication.
* Supports broadcasting one notification to thousands of students.
* Allows every student to have independent read/unread status.
* Improves maintainability.
* Reduces storage requirements.

---

# Indexing Strategy

The following indexes are recommended to optimize query performance.

## Primary Indexes

```sql
PRIMARY KEY (user_id)

PRIMARY KEY (notification_id)

PRIMARY KEY (id)
```

---

## Secondary Indexes

```sql
CREATE INDEX idx_user_notifications_user
ON user_notifications(user_id);

CREATE INDEX idx_user_notifications_read
ON user_notifications(user_id,is_read);

CREATE INDEX idx_notifications_created
ON notifications(created_at DESC);

CREATE INDEX idx_notifications_type
ON notifications(notification_type);
```

### Purpose of Indexes

| Index             | Purpose                                    |
| ----------------- | ------------------------------------------ |
| user_id           | Fetch notifications of one student quickly |
| user_id + is_read | Fast unread notification lookup            |
| created_at        | Fast sorting by latest notifications       |
| notification_type | Efficient filtering                        |

---

# SQL Queries

## Fetch Notifications

```sql
SELECT
n.notification_id,
n.title,
n.message,
n.notification_type,
u.is_read,
n.created_at
FROM notifications n
JOIN user_notifications u
ON n.notification_id = u.notification_id
WHERE u.user_id = ?
ORDER BY n.created_at DESC
LIMIT ? OFFSET ?;
```

---

## Fetch Notification by ID

```sql
SELECT
n.notification_id,
n.title,
n.message,
n.notification_type,
u.is_read,
u.read_at
FROM notifications n
JOIN user_notifications u
ON n.notification_id = u.notification_id
WHERE u.user_id = ?
AND n.notification_id = ?;
```

---

## Mark Notification as Read

```sql
UPDATE user_notifications
SET
is_read = TRUE,
read_at = CURRENT_TIMESTAMP
WHERE
user_id = ?
AND notification_id = ?;
```

---

## Mark All Notifications as Read

```sql
UPDATE user_notifications
SET
is_read = TRUE,
read_at = CURRENT_TIMESTAMP
WHERE
user_id = ?
AND is_read = FALSE;
```

---

## Get Unread Notification Count

```sql
SELECT COUNT(*)
FROM user_notifications
WHERE
user_id = ?
AND is_read = FALSE;
```

---

## Create Notification

```sql
INSERT INTO notifications
(
notification_id,
title,
message,
notification_type,
priority
)
VALUES
(
?,
?,
?,
?,
?
);
```

---

## Assign Notification to Students

```sql
INSERT INTO user_notifications
(
id,
user_id,
notification_id
)
VALUES
(
?,
?,
?
);
```

---

# Scalability Challenges

As the number of students and notifications increases, the following issues may occur.

### 1. Large Notification Table

Millions of notifications will increase query execution time.

### 2. Slow Sorting

Ordering notifications by creation time becomes expensive.

### 3. Slow Filtering

Searching unread notifications without indexes leads to full table scans.

### 4. High Read Traffic

Students frequently open the notification page, increasing database load.

### 5. Bulk Notification Delivery

Placement notifications may be sent to tens of thousands of students simultaneously.

### 6. Storage Growth

Historical notifications continuously increase storage usage.

---

# Proposed Solutions

## Proper Indexing

Indexes significantly reduce lookup time for frequently executed queries.

---

## Pagination

Use `page` and `limit` parameters to retrieve only a subset of notifications.

---

## Read Replicas

Direct read-heavy APIs such as fetching notifications and unread counts to read replicas while writes continue on the primary database.

---

## Redis Cache

Cache frequently requested values such as unread notification count.

---

## Database Partitioning

Partition notifications by creation date (monthly or yearly) to reduce scan size.

---

## Background Workers

Bulk notification assignment should be processed asynchronously using worker processes instead of blocking API requests.

---

## Archiving

Move notifications older than a defined retention period (for example, one year) into archive tables to reduce active table size.

---

# Design Decisions

The database design follows normalization principles to eliminate redundancy while maintaining flexibility and scalability.

The `User_Notifications` table separates notification content from user-specific read status, enabling one notification to be delivered to many students without duplicating data.

Composite indexes are introduced to optimize the most frequently executed queries, particularly unread notification retrieval and sorting by creation time.

The architecture supports future enhancements such as Redis caching, read replicas, database partitioning, and asynchronous processing without requiring major schema changes.

This design aligns with the REST APIs defined in Stage 1 and provides a scalable, maintainable, and production-ready foundation for the Campus Notification Platform.
--- 


# Stage 3

## Query Analysis

The existing query used to fetch unread notifications is shown below:

```sql
SELECT *
FROM notifications
WHERE studentID = 1042
AND isRead = false
ORDER BY createdAt ASC;
```

---

# Is the Query Accurate?

The query is functionally correct because it retrieves all unread notifications for a specific student and sorts them by creation time.

However, from a performance and scalability perspective, it is **not optimal** for a production database containing approximately **50,000 students** and **5 million notifications**.

---

# Why is this Query Slow?

As the notification table grows, several performance bottlenecks appear.

## 1. Full Table Scan

Without an appropriate index, the database scans millions of records to locate notifications belonging to a single student.

This increases query execution time significantly.

---

## 2. Expensive Sorting

The query sorts the filtered notifications using

```sql
ORDER BY createdAt ASC
```

If no index exists on the sorting column, the database performs an additional sorting operation.

---

## 3. SELECT *

The query retrieves every column from the table.

In most cases, the frontend requires only a subset of fields such as

* notification_id
* title
* message
* notification_type
* created_at

Reading unnecessary columns increases disk I/O and memory usage.

---

## 4. Large Dataset

With millions of notifications, sequential scans become increasingly expensive, resulting in slower response times.

---

# Optimized Query

Instead of selecting every column, retrieve only the required fields.

```sql
SELECT
notification_id,
title,
message,
notification_type,
created_at
FROM notifications
WHERE studentID = ?
AND isRead = FALSE
ORDER BY createdAt DESC;
```

Using descending order ensures the newest notifications appear first, which aligns with typical notification system behavior.

---

# Recommended Index

The most appropriate index is a composite index.

```sql
CREATE INDEX idx_student_read_created
ON notifications(studentID, isRead, createdAt DESC);
```

---

# Why This Composite Index?

The database filters by

```text
studentID
```

then

```text
isRead
```

and finally sorts by

```text
createdAt
```

The composite index follows exactly the same order as the query execution.

This allows the database to:

* Locate the student's records quickly.
* Filter unread notifications efficiently.
* Return results already sorted without an additional sort operation.

---

# Computational Cost

## Without Index

The database may perform a sequential scan across the notification table.

Approximate complexity:

```
O(N)
```

where

```
N = Total Notifications
```

Sorting may further increase the cost to

```
O(N log N)
```

---

## With Composite Index

The lookup becomes an indexed search.

Approximate complexity:

```
O(log N + K)
```

where

* N = Total notifications
* K = Matching unread notifications

This significantly reduces execution time.

---

# Should Every Column Be Indexed?

No.

Creating indexes on every column is not recommended.

## Reasons

### Increased Storage

Every index occupies additional disk space.

---

### Slower Inserts

Whenever a new notification is inserted, all related indexes must also be updated.

This increases write latency.

---

### Slower Updates

Updating indexed columns requires updating every associated index.

---

### Higher Maintenance Cost

Indexes require periodic maintenance and rebuilding.

Too many indexes negatively impact overall database performance.

---

## Best Practice

Only create indexes for

* Frequently filtered columns
* Frequently joined columns
* Frequently sorted columns

---

# Query to Find Students Who Received Placement Notifications in the Last 7 Days

Assuming the normalized schema from Stage 2.

```sql
SELECT DISTINCT
u.user_id,
u.full_name,
u.email
FROM users u
JOIN user_notifications un
ON u.user_id = un.user_id
JOIN notifications n
ON n.notification_id = un.notification_id
WHERE
n.notification_type = 'Placement'
AND n.created_at >= CURRENT_DATE - INTERVAL '7 days';
```

---

# Additional Optimization Techniques

## Pagination

Never fetch all notifications at once.

Instead use

```sql
LIMIT 20 OFFSET 0;
```

---

## Read Replicas

Route read-heavy APIs such as notification retrieval to read replicas while writes continue on the primary database.

---

## Redis Cache

Frequently accessed information such as unread notification counts can be cached to reduce database load.

---

## Table Partitioning

Partition the notification table by creation date to reduce scan size as historical data grows.

---

## Covering Index

If the database supports covering indexes, include frequently returned columns in the index to reduce table lookups.

---

# Design Decisions

The optimized solution replaces a sequential scan with a composite index that matches the query pattern.

Selecting only the required columns minimizes disk I/O and improves response time.

Composite indexing provides efficient filtering and sorting while avoiding unnecessary indexes that would negatively impact insert and update performance.

This approach supports millions of notification records while maintaining fast query execution and aligns with the database schema designed in Stage 2.

