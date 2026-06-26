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