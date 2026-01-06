# 🚀 Deployment Guide - Backend Anime Platform

## Quick Start

### Prerequisites
- Python 3.11+
- PostgreSQL 15+
- Redis 7+
- Docker & Docker Compose (optional)

### Local Development

1. **Install dependencies:**
```bash
pip install -e .
```

2. **Set up environment:**
```bash
cp .env.example .env
# Edit .env with your configuration
```

3. **Run database migrations:**
```bash
alembic upgrade head
```

4. **Start the server:**
```bash
uvicorn src.api.main:app --reload --host 0.0.0.0 --port 8000
```

### Docker Deployment

```bash
docker-compose up -d
```

This will start:
- PostgreSQL database
- Redis cache
- Backend API server
- Celery worker

## API Endpoints

### Health Checks
- `GET /api/v1/health` - Basic health check
- `GET /api/v1/health/db` - Database health check
- `GET /api/v1/health/ready` - Kubernetes readiness probe
- `GET /api/v1/health/live` - Kubernetes liveness probe

### Authentication
- `POST /api/v1/auth/register` - Register new user
- `POST /api/v1/auth/login` - User login
- `POST /api/v1/auth/logout` - User logout
- `GET /api/v1/auth/me` - Get current user

### Releases
- `GET /api/v1/releases` - List releases (paginated)
- `GET /api/v1/releases/{id}` - Get release by ID
- `POST /api/v1/releases` - Create release (admin)
- `PUT /api/v1/releases/{id}` - Update release (admin)
- `DELETE /api/v1/releases/{id}` - Delete release (admin)
- `POST /api/v1/releases/search` - Search releases

### Episodes
- `GET /api/v1/releases/{id}/episodes` - List episodes for release
- `GET /api/v1/episodes/{id}` - Get episode by ID
- `POST /api/v1/releases/{id}/episodes` - Create episode (admin)
- `DELETE /api/v1/episodes/{id}` - Delete episode (admin)

### Favorites
- `GET /api/v1/favorites` - Get user favorites
- `POST /api/v1/favorites/{release_id}` - Add to favorites
- `DELETE /api/v1/favorites/{release_id}` - Remove from favorites

## API Documentation

Interactive API documentation is available at:
- Swagger UI: `http://localhost:8000/api/docs`
- ReDoc: `http://localhost:8000/api/redoc`

## Configuration

Environment variables (see `.env.example`):

```bash
# Database
DATABASE_URL=postgresql+asyncpg://user:password@localhost/dbname

# Redis
REDIS_URL=redis://localhost:6379/0

# App
APP_NAME=Backend Anime Platform
APP_VERSION=1.0.0
DEBUG=false

# Security
SECRET_KEY=your-secret-key-here
CORS_ORIGINS=["http://localhost:3000"]

# Celery
CELERY_BROKER_URL=redis://localhost:6379/1
CELERY_RESULT_BACKEND=redis://localhost:6379/2
```

## Database Migrations

```bash
# Create new migration
alembic revision --autogenerate -m "Description"

# Apply migrations
alembic upgrade head

# Rollback migration
alembic downgrade -1
```

## Testing

```bash
# Run all tests
pytest

# Run with coverage
pytest --cov=src --cov-report=html

# Run specific test file
pytest tests/unit/test_domain.py
```

## Production Considerations

1. **Security:**
   - Change `SECRET_KEY` to a strong random value
   - Use HTTPS in production
   - Enable rate limiting
   - Configure proper CORS origins

2. **Performance:**
   - Enable Redis caching
   - Configure connection pooling
   - Set up CDN for static files
   - Use production ASGI server (e.g., Gunicorn + Uvicorn)

3. **Monitoring:**
   - Set up logging (ELK stack, CloudWatch, etc.)
   - Configure application metrics (Prometheus)
   - Set up error tracking (Sentry)
   - Monitor health check endpoints

4. **Scaling:**
   - Horizontal scaling with load balancer
   - Database read replicas
   - Redis cluster for high availability
   - Celery workers for background tasks

## Kubernetes Deployment

Example deployment manifest:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-ani
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend-ani
  template:
    metadata:
      labels:
        app: backend-ani
    spec:
      containers:
      - name: api
        image: backend-ani:latest
        ports:
        - containerPort: 8000
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
        livenessProbe:
          httpGet:
            path: /api/v1/health/live
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /api/v1/health/ready
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5
```

## Troubleshooting

### Database Connection Issues
```bash
# Check PostgreSQL is running
pg_isready -h localhost -p 5432

# Check connection from app
psql $DATABASE_URL
```

### Redis Connection Issues
```bash
# Check Redis is running
redis-cli ping

# Check connection
redis-cli -u $REDIS_URL ping
```

### Migration Issues
```bash
# Check current migration
alembic current

# Show migration history
alembic history

# Stamp database with specific version
alembic stamp head
```

## Support

For issues and questions:
- Check documentation in `/docs`
- Review architecture in `ARCHITECTURE_SUMMARY.md`
- See implementation details in `IMPLEMENTATION_COMPLETE.md`
