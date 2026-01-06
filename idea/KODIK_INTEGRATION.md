# Kodik Parser Integration Guide

## Overview

The Kodik Parser Adapter provides a complete integration with the Kodik API for parsing and importing anime content into the platform. It follows Clean Architecture principles and integrates seamlessly with existing domain entities, use cases, and repositories.

## Architecture

```
┌─────────────────────────────────────────────────────┐
│              API Layer (FastAPI)                    │
│  /api/v1/parser/kodik/* endpoints                   │
└──────────────────┬──────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────┐
│         Infrastructure Layer                        │
│  KodikAdapter (parser adapter)                      │
│  Celery Tasks (background processing)               │
└──────────────────┬──────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────┐
│         Application Layer                           │
│  Use Cases: CreateRelease, CreateEpisode,           │
│  AttachVideoSource, UpdateMetadata                  │
└──────────────────┬──────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────┐
│         Domain Layer                                │
│  Entities: Release, Episode, VideoSource            │
│  Events: ReleaseCreated, EpisodeAdded, etc.         │
└─────────────────────────────────────────────────────┘
```

## Features

### ✅ Implemented

1. **Async HTTP Requests**
   - Non-blocking aiohttp client
   - Exponential backoff retry logic
   - Configurable timeout and max retries
   - Connection pooling

2. **Data Parsing**
   - Release metadata (title, description, year, genres)
   - Episode information (number, title, duration)
   - Video sources (HLS, MP4, DASH) with multiple qualities
   - Image URLs (posters, screenshots)
   - Ratings from Shikimori

3. **Domain Entity Conversion**
   - Kodik JSON → Domain Entities (Release, Episode, VideoSource)
   - Automatic quality mapping (360p → 4K)
   - Format detection (HLS, DASH, MP4)
   - Status mapping (ongoing, completed, announced)

4. **Video Proxy**
   - Ad-free video URLs via ProxyService
   - Transparent URL rewriting
   - Client-side compatibility

5. **Batch Processing**
   - Configurable batch size
   - Async batch imports
   - Progress logging
   - Error handling per title

6. **Caching**
   - Redis cache integration
   - Configurable TTL (default: 1 hour)
   - Cache key generation
   - Automatic cache invalidation

7. **Background Tasks**
   - Celery integration
   - Single title parsing task
   - Batch parsing task
   - Scheduled updates (every 6 hours)

8. **API Endpoints**
   - POST `/api/v1/parser/kodik/parse` - Parse single title
   - POST `/api/v1/parser/kodik/batch` - Batch import
   - GET `/api/v1/parser/kodik/search` - Search Kodik API

## Usage

### Configuration

Add to `.env`:

```env
# Kodik API
KODIK_API_TOKEN=your_kodik_api_token_here
KODIK_API_URL=https://kodikapi.com

# Parser Settings
PARSER_BATCH_SIZE=50
PARSER_TIMEOUT=30
PARSER_MAX_RETRIES=3

# Redis (for caching)
REDIS_URL=redis://localhost:6379/0

# Celery (for background tasks)
CELERY_BROKER_URL=redis://localhost:6379/1
CELERY_RESULT_BACKEND=redis://localhost:6379/2
```

### API Usage Examples

#### 1. Search for Anime

```bash
curl -X GET "http://localhost:8000/api/v1/parser/kodik/search?query=One+Piece&limit=10" \
  -H "Authorization: Bearer YOUR_ADMIN_TOKEN"
```

Response:
```json
{
  "query": "One Piece",
  "total": 150,
  "results": [
    {
      "id": "12345",
      "title": "One Piece",
      "title_orig": "ワンピース",
      "year": 1999,
      "type": "tv-series",
      "poster_url": "https://...",
      "rating": "8.9"
    }
  ]
}
```

#### 2. Parse Single Title

```bash
curl -X POST "http://localhost:8000/api/v1/parser/kodik/parse" \
  -H "Authorization: Bearer YOUR_ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"kodik_id": "12345"}'
```

Response:
```json
{
  "status": "success",
  "release_id": "uuid-here",
  "message": "Successfully parsed Kodik ID 12345"
}
```

#### 3. Batch Import

```bash
curl -X POST "http://localhost:8000/api/v1/parser/kodik/batch" \
  -H "Authorization: Bearer YOUR_ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"kodik_ids": ["12345", "67890", "11111"]}'
```

Response:
```json
{
  "status": "success",
  "requested": 3,
  "imported": 3,
  "release_ids": ["uuid1", "uuid2", "uuid3"],
  "message": "Imported 3/3 titles"
}
```

### Python Usage

```python
from src.infrastructure.parsers.kodik import KodikAdapter, KodikConfig
from src.infrastructure.database.uow import SQLAlchemyUnitOfWork
from src.infrastructure.events.bus import get_event_bus
from src.infrastructure.cache.redis import RedisCache

# Configure
config = KodikConfig(
    api_token="your_token",
    timeout=30,
    max_retries=3,
    batch_size=50,
)

# Initialize components
uow = SQLAlchemyUnitOfWork(session)
event_bus = get_event_bus()
cache = RedisCache(redis_url)

# Use adapter
async with KodikAdapter(config, uow, event_bus, cache) as adapter:
    # Parse single title
    release_id = await adapter.parse_and_save_title("kodik_id")
    
    # Batch import
    release_ids = await adapter.batch_import(["id1", "id2", "id3"])
    
    # Fetch and parse manually
    data = await adapter.fetch_json("search", params={"id": "kodik_id"})
    kodik_release = adapter.parse_release(data["results"][0])
    release, episodes, videos = await adapter.to_domain_entities(kodik_release)
    await adapter.save(release, episodes, videos)
```

### Celery Tasks

```python
from src.infrastructure.parsers.tasks import (
    parse_kodik_title_task,
    batch_parse_kodik_task,
    scheduled_parse_updates_task,
)

# Queue single parse
result = parse_kodik_title_task.delay("kodik_id")
release_uuid = result.get()  # Blocking wait

# Queue batch parse
result = batch_parse_kodik_task.delay(["id1", "id2", "id3"])
release_uuids = result.get()

# Scheduled task runs automatically every 6 hours
```

## Data Flow

1. **HTTP Request** → Kodik API
2. **JSON Response** → KodikAdapter.fetch_json()
3. **Parse** → KodikAdapter.parse_release/episodes/videos()
4. **Convert** → Domain Entities (Release, Episode, VideoSource)
5. **Proxy** → ProxyService generates ad-free URLs
6. **Save** → Use Cases (CreateRelease, CreateEpisode, AttachVideoSource)
7. **Events** → Domain Events published (ReleaseCreated, EpisodeAdded)
8. **Cache** → Results cached in Redis

## Video Quality Mapping

| Kodik | Domain Enum | Resolution |
|-------|-------------|------------|
| 360p | Q_360P | 640×360 |
| 480p | Q_480P | 854×480 |
| 720p | Q_720P | 1280×720 |
| 1080p | Q_1080P | 1920×1080 |
| 2160p | Q_4K | 3840×2160 |

## Format Detection

- `.m3u8` or `hls` in URL → VideoFormat.HLS
- `.mpd` or `dash` in URL → VideoFormat.DASH
- `.mp4` in URL → VideoFormat.MP4
- Default → VideoFormat.HLS

## Error Handling

- **HTTP Errors**: Automatic retry with exponential backoff
- **Parse Errors**: Logged, continue with next title in batch
- **Validation Errors**: Proper error messages returned to API
- **Database Errors**: Transaction rollback via Unit of Work

## Performance Considerations

### Async Operations
- All HTTP requests are non-blocking
- Batch processing uses asyncio.gather()
- Database operations use async SQLAlchemy

### Batching
- Configurable batch size (default: 50)
- Prevents overwhelming weak servers
- Progress logging per batch

### Caching
- API responses cached for 1 hour
- Reduces Kodik API load
- Faster repeated requests

### Background Processing
- Celery for long-running tasks
- Non-blocking API responses
- Scheduled updates

## Testing

```python
import pytest
from src.infrastructure.parsers.kodik import KodikAdapter, KodikConfig

@pytest.mark.asyncio
async def test_parse_release():
    # Mock Kodik API response
    mock_data = {
        "title": "Test Anime",
        "year": 2024,
        "material_data": {
            "description": "Test description",
            "anime_status": "ongoing",
        }
    }
    
    adapter = KodikAdapter(config, uow, event_bus)
    kodik_release = adapter.parse_release(mock_data)
    
    assert kodik_release.title_ru == "Test Anime"
    assert kodik_release.year == 2024
    assert kodik_release.status == "ongoing"
```

## Deployment

### Docker Compose

```yaml
services:
  celery_worker:
    build: .
    command: celery -A src.infrastructure.parsers.tasks worker --loglevel=info
    environment:
      - KODIK_API_TOKEN=${KODIK_API_TOKEN}
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=${REDIS_URL}
    depends_on:
      - redis
      - postgres

  celery_beat:
    build: .
    command: celery -A src.infrastructure.parsers.tasks beat --loglevel=info
    environment:
      - KODIK_API_TOKEN=${KODIK_API_TOKEN}
    depends_on:
      - redis
```

### Production Checklist

- [ ] Set KODIK_API_TOKEN in environment
- [ ] Configure Redis for caching
- [ ] Configure Celery broker/backend
- [ ] Set appropriate batch size for server capacity
- [ ] Enable structured logging
- [ ] Monitor parser execution logs
- [ ] Set up alerts for parse failures
- [ ] Configure proxy service for ad removal
- [ ] Test with sample Kodik IDs
- [ ] Verify video playback works

## Monitoring

### Logs

```python
from src.infrastructure.logging.logger import get_logger

logger = get_logger(__name__)
logger.info(f"Parsing Kodik ID: {kodik_id}")
logger.error(f"Parse failed: {error}")
```

### Metrics

- Releases parsed per hour
- Episodes imported per hour
- API request success rate
- Cache hit rate
- Average parse duration

## Limitations

1. **No Direct Copy** - All code is original, no copying from DLE/Kodik/AnimeParsers
2. **Domain Immutable** - Cannot modify existing domain entities
3. **API Rate Limits** - Respect Kodik API limits
4. **Server Capacity** - Batch size should match server specs

## Future Enhancements

- [ ] Genre mapping from Kodik to domain genres
- [ ] Audio track and subtitle parsing
- [ ] Automated quality upgrades
- [ ] Delta updates (only changed titles)
- [ ] Multi-source parsing (not just Kodik)
- [ ] Advanced error recovery
- [ ] Parse analytics dashboard

## Support

For issues or questions:
- Check logs: `src/infrastructure/parsers/`
- Review API response in `/api/v1/parser/kodik/search`
- Test with single title before batch
- Verify Kodik API token is valid

## License

This implementation is proprietary and does not copy code from:
- DLE (DataLife Engine)
- Kodik official clients
- AnimeParsers repository

All parsing logic is original and integrates with our Clean Architecture.
