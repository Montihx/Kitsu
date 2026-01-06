# Phase 5 API Documentation

## Overview

Phase 5 adds Analytics, CMS (Content Management System), and Subscriptions/Notifications to the platform. All features are optimized for weak servers with async operations, Redis caching, and rate limiting.

---

## Analytics Endpoints

### GET /api/v1/analytics/views

Get view analytics for releases.

**Public endpoint** - No authentication required.

**Query Parameters:**
- `release_ids` (optional): List of release UUIDs to filter
- `limit` (optional): Number of releases to return (default: 10, max: 100)

**Response:**
```json
{
  "releases": [
    {
      "release_id": "uuid",
      "release_title": "Anime Title",
      "total_views": 1500,
      "unique_viewers": 850,
      "average_completion": 75.5
    }
  ],
  "total_releases": 1
}
```

**Caching:** 10 minutes TTL in Redis

---

### GET /api/v1/analytics/top

Get top releases and genres by popularity.

**Public endpoint** - No authentication required.

**Query Parameters:**
- `limit` (optional): Number of items to return (default: 10, max: 50)

**Response:**
```json
{
  "top_releases": [
    {
      "release_id": "uuid",
      "release_title": "Popular Anime",
      "total_views": 5000,
      "unique_viewers": 2500,
      "average_rating": 9.2,
      "popularity_score": 87.5
    }
  ],
  "top_genres": [
    {
      "genre_id": "uuid",
      "genre_name": "Action",
      "total_views": 15000,
      "release_count": 25
    }
  ]
}
```

**Caching:** 1 hour TTL in Redis

---

## CMS (Content Management) Endpoints

All CMS endpoints require **ADMIN** role authentication.

All operations return **202 Accepted** status to indicate async processing.

### POST /api/v1/admin/releases

Create a new release.

**Authentication:** Admin only

**Request Body:**
```json
{
  "title": "New Anime Title",
  "description": "Anime description",
  "genre_ids": ["uuid1", "uuid2"],
  "franchise_id": "uuid",
  "release_date": "2024-01-01T00:00:00Z",
  "status": "announced",
  "tags": ["tag1", "tag2"],
  "poster_url": "https://cdn.example.com/poster.jpg",
  "banner_url": "https://cdn.example.com/banner.jpg",
  "total_episodes": 12
}
```

**Response (202 Accepted):**
```json
{
  "id": "uuid",
  "status": "accepted",
  "message": "Release creation initiated"
}
```

---

### PUT /api/v1/admin/releases/{release_id}

Update an existing release.

**Authentication:** Admin only

**Request Body:** (all fields optional)
```json
{
  "title": "Updated Title",
  "description": "Updated description",
  "status": "ongoing"
}
```

**Response (202 Accepted):**
```json
{
  "id": "uuid",
  "status": "accepted",
  "message": "Release update initiated"
}
```

---

### DELETE /api/v1/admin/releases/{release_id}

Delete a release.

**Authentication:** Admin only

**Response (202 Accepted):**
```json
{
  "id": "uuid",
  "status": "accepted",
  "message": "Release deletion initiated"
}
```

---

### POST /api/v1/admin/episodes

Create a new episode.

**Authentication:** Admin only

**Request Body:**
```json
{
  "release_id": "uuid",
  "title": "Episode 1 Title",
  "number": 1,
  "duration": 1440,
  "description": "Episode description",
  "release_date": "2024-01-01T00:00:00Z",
  "status": "upcoming",
  "thumbnail_url": "https://cdn.example.com/thumb.jpg"
}
```

**Response (202 Accepted):**
```json
{
  "id": "uuid",
  "status": "accepted",
  "message": "Episode creation initiated"
}
```

---

### PUT /api/v1/admin/episodes/{episode_id}

Update an existing episode.

**Authentication:** Admin only

**Request Body:** (all fields optional)
```json
{
  "title": "Updated Episode Title",
  "status": "released"
}
```

**Response (202 Accepted):**
```json
{
  "id": "uuid",
  "status": "accepted",
  "message": "Episode update initiated"
}
```

---

### DELETE /api/v1/admin/episodes/{episode_id}

Delete an episode.

**Authentication:** Admin only

**Response (202 Accepted):**
```json
{
  "id": "uuid",
  "status": "accepted",
  "message": "Episode deletion initiated"
}
```

---

## Subscriptions Endpoints

### POST /api/v1/subscriptions

Subscribe to a release, genre, or franchise.

**Authentication:** Required

**Request Body:**
```json
{
  "subscription_type": "release",
  "target_id": "uuid"
}
```

**Valid subscription_type values:**
- `release` - Subscribe to new episodes of a specific release
- `genre` - Subscribe to new releases in a genre
- `franchise` - Subscribe to new releases in a franchise

**Response (201 Created):**
```json
{
  "id": "uuid",
  "user_id": "uuid",
  "subscription_type": "release",
  "target_id": "uuid",
  "created_at": "2024-01-01T00:00:00Z",
  "is_active": true
}
```

---

### GET /api/v1/subscriptions

List all subscriptions for the current user.

**Authentication:** Required

**Query Parameters:**
- `skip` (optional): Offset for pagination (default: 0)
- `limit` (optional): Number of items to return (default: 50, max: 100)

**Response:**
```json
{
  "subscriptions": [
    {
      "id": "uuid",
      "user_id": "uuid",
      "subscription_type": "release",
      "target_id": "uuid",
      "created_at": "2024-01-01T00:00:00Z",
      "is_active": true
    }
  ],
  "total": 1
}
```

---

### DELETE /api/v1/subscriptions/{subscription_id}

Unsubscribe from a release, genre, or franchise.

**Authentication:** Required (user can only delete their own subscriptions)

**Response:** 204 No Content

---

## Notifications Endpoints

### GET /api/v1/notifications

List notifications for the current user.

**Authentication:** Required

**Query Parameters:**
- `unread_only` (optional): Show only unread notifications (default: false)
- `skip` (optional): Offset for pagination (default: 0)
- `limit` (optional): Number of items to return (default: 50, max: 100)

**Response:**
```json
{
  "notifications": [
    {
      "id": "uuid",
      "notification_type": "new_episode",
      "title": "New Episode Released!",
      "message": "A new episode 'Episode 5' is now available to watch.",
      "related_id": "uuid",
      "is_read": false,
      "created_at": "2024-01-01T00:00:00Z",
      "read_at": null
    }
  ],
  "total": 1,
  "unread_count": 1
}
```

**Notification Types:**
- `new_episode` - New episode released for subscribed release
- `new_release` - New release added in subscribed genre
- `recommendation` - Personalized recommendation

---

### POST /api/v1/notifications/{notification_id}/read

Mark a notification as read.

**Authentication:** Required

**Response:** 204 No Content

---

### GET /api/v1/notifications/unread/count

Get count of unread notifications.

**Authentication:** Required

**Response:**
```json
{
  "unread_count": 5
}
```

---

## Rate Limiting

### Notifications

- **Maximum 5 notifications per hour per user**
- Rate limit is enforced when creating notifications via Celery tasks
- If rate limit is exceeded, notification creation is skipped

### Old Notification Cleanup

- Read notifications older than 30 days are automatically deleted
- Cleanup runs via Celery scheduled task

---

## Celery Tasks

### send_new_episode_notification_task

Triggered when a new episode is created. Sends notifications to all users subscribed to the release.

**Parameters:**
- `release_id`: UUID of the release
- `episode_id`: UUID of the new episode
- `episode_title`: Title of the episode

**Rate Limit:** 5 notifications per hour per user

---

### send_new_release_notification_task

Triggered when a new release is added. Sends notifications to all users subscribed to the release's genres.

**Parameters:**
- `release_id`: UUID of the new release
- `release_title`: Title of the release
- `genre_ids`: List of genre UUIDs

**Rate Limit:** 5 notifications per hour per user

---

### send_recommendation_notification_task

Send a personalized recommendation to a user.

**Parameters:**
- `user_id`: UUID of the user
- `release_id`: UUID of the recommended release
- `release_title`: Title of the release
- `reason`: Reason for recommendation

**Rate Limit:** 5 notifications per hour per user

---

### cleanup_old_notifications_task

Delete read notifications older than N days (default: 30).

**Parameters:**
- `days`: Age threshold for deletion (default: 30)

**Returns:** Number of notifications deleted

---

## Database Schema

### view_metrics

Detailed view tracking for analytics.

| Column | Type | Description |
|--------|------|-------------|
| id | UUID | Primary key |
| release_id | UUID | Release being viewed |
| episode_id | UUID | Episode being viewed (optional) |
| user_id | UUID | User viewing (optional, for anonymous tracking) |
| viewed_at | DateTime | Timestamp of view |
| session_id | String | Session identifier (optional) |
| duration_seconds | Integer | Duration of viewing |
| completion_percentage | Float | Percentage of content watched |

**Indexes:**
- release_id
- episode_id
- user_id
- viewed_at
- (release_id, viewed_at) - Composite

---

### popularity_metrics

Aggregated popularity metrics for releases.

| Column | Type | Description |
|--------|------|-------------|
| id | UUID | Primary key |
| release_id | UUID | Release (unique) |
| total_views | Integer | Total view count |
| unique_viewers | Integer | Unique viewer count |
| average_rating | Float | Average user rating |
| popularity_score | Float | Calculated popularity score |
| last_calculated_at | DateTime | Last calculation time |
| updated_at | DateTime | Last update time |

**Indexes:**
- release_id (unique)
- popularity_score
- (popularity_score, total_views) - Composite

---

### subscriptions

User subscriptions to releases, genres, or franchises.

| Column | Type | Description |
|--------|------|-------------|
| id | UUID | Primary key |
| user_id | UUID | Subscriber user |
| subscription_type | String | Type: release, genre, franchise |
| target_id | UUID | ID of subscribed entity |
| created_at | DateTime | Subscription creation time |
| is_active | Boolean | Active status |

**Indexes:**
- user_id
- target_id
- (user_id, subscription_type, target_id) - Unique composite

---

### notifications

User notifications.

| Column | Type | Description |
|--------|------|-------------|
| id | UUID | Primary key |
| user_id | UUID | Recipient user |
| notification_type | String | Type: new_episode, new_release, recommendation |
| title | String | Notification title |
| message | Text | Notification message |
| related_id | UUID | Related entity ID (optional) |
| is_read | Boolean | Read status |
| created_at | DateTime | Creation time |
| read_at | DateTime | Time marked as read (optional) |

**Indexes:**
- user_id
- created_at
- (user_id, is_read, created_at) - Composite

---

## Performance Optimizations

### Caching Strategy

- **Analytics views:** 10 minutes TTL
- **Analytics top:** 1 hour TTL
- Automatic fallback to database if cache unavailable

### Async Operations

- All CMS operations return 202 Accepted immediately
- Heavy operations are processed asynchronously
- No blocking database operations in API layer

### Batch Updates

- Bulk operations use batch processing
- Minimize database round-trips
- Transaction boundaries carefully managed

---

## Error Responses

### 401 Unauthorized

Authentication required but not provided.

```json
{
  "detail": "Not authenticated"
}
```

### 403 Forbidden

Insufficient permissions.

```json
{
  "detail": "Insufficient permissions"
}
```

### 404 Not Found

Resource not found.

```json
{
  "detail": "Release not found"
}
```

### 409 Conflict

Resource already exists (e.g., duplicate subscription).

```json
{
  "detail": "Subscription already exists"
}
```

### 500 Internal Server Error

Server error.

```json
{
  "detail": "Internal server error"
}
```
