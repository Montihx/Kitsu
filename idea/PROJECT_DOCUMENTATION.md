# Backend Ani - Anime Platform

Full-stack anime platform backend built with Clean Architecture principles.

## 🎯 Почему `src/` структура? (Why src/ layout?)

### Современный профессиональный стандарт (Modern Professional Standard)

Структура `src/` является **индустриальным стандартом** для профессиональных Python проектов:

**✅ Преимущества src/ layout:**

1. **Изоляция кода от тестов** - Чёткое разделение между production кодом и тестами
2. **Правильный import path** - Избегаем проблем с `sys.path` и относительными импортами
3. **Packaging ready** - Код готов к публикации в PyPI без модификаций
4. **Editable installs** - `pip install -e .` работает корректно
5. **Защита от ошибок** - Невозможно случайно импортировать тесты в production
6. **Стандарт больших проектов** - Используется в Django, FastAPI examples, Flask, и других enterprise проектах

**Примеры крупных проектов с src/ layout:**
- [FastAPI](https://github.com/tiangolo/fastapi) - официальные примеры
- [Django](https://github.com/django/django) - структура приложений
- [Poetry](https://github.com/python-poetry/poetry) - сам Poetry использует src/
- [Black](https://github.com/psf/black) - code formatter
- [Requests](https://github.com/psf/requests) - популярная HTTP библиотека

### Альтернативная плоская структура (Flat layout)

❌ **Плоская структура БЕЗ src/ имеет проблемы:**
```
backend-ani/
├── domain/          # ⚠️ Конфликт с системными модулями
├── application/     # ⚠️ Может быть импортирован из tests/
├── tests/           # ⚠️ Рядом с production кодом
```

**Проблемы:**
- Тесты могут случайно импортировать не установленный код
- Путаница с PYTHONPATH
- Сложности при packaging
- Не подходит для больших команд

### Наша архитектура - Clean Architecture

```
backend-ani/
├── src/                    # 👈 Весь application код здесь
│   ├── domain/            # 🎯 Чистая бизнес-логика (CORE)
│   ├── application/       # 🎯 Use cases и оркестрация
│   ├── infrastructure/    # 🎯 БД, кэш, внешние сервисы
│   └── api/              # 🎯 HTTP endpoints и схемы
├── config/                # ⚙️ Конфигурация
├── migrations/            # 📊 Alembic миграции БД
├── tests/                 # 🧪 Тесты изолированы
├── docker-compose.yml     # 🐳 Docker services
└── pyproject.toml        # 📦 Зависимости
```

**Почему так удобно:**
1. Файлы легко найти - всё в `src/domain/`, `src/api/`, etc.
2. Импорты понятные - `from src.domain.anime import Release`
3. IDE автодополнение работает идеально
4. Нет конфликтов имён с системными модулями
5. Scalable - можно добавлять новые домены без путаницы

## 📦 Навигация по проекту

### Где что искать:

| Что нужно | Где искать | Пример |
|-----------|-----------|---------|
| **Бизнес-логика** | `src/domain/` | `src/domain/anime/entities.py` |
| **Use cases** | `src/application/` | `src/application/anime/use_cases.py` |
| **API endpoints** | `src/api/v1/routes/` | `src/api/v1/routes/releases.py` |
| **Модели БД** | `src/infrastructure/database/models/` | `src/infrastructure/database/models/anime.py` |
| **Схемы API** | `src/api/schemas/` | `src/api/schemas/anime.py` |
| **Конфигурация** | `config/` | `config/settings.py` |
| **Миграции** | `migrations/` | `migrations/versions/` |
| **Тесты** | `tests/` | `tests/unit/test_domain.py` |

### Быстрый поиск файлов:

```bash
# Найти все entities
find src/domain -name "entities.py"

# Найти все use cases
find src/application -name "use_cases.py"

# Найти API routes
find src/api -name "*.py" -path "*/routes/*"

# Найти модели БД
find src/infrastructure/database/models -name "*.py"
```

## Project Structure

```
backend-ani/
├── src/
│   ├── domain/           # Domain layer (entities, value objects, events, services)
│   │   ├── accounts/     # User accounts domain
│   │   ├── anime/        # Anime content domain
│   │   ├── media/        # Media and streaming domain
│   │   └── teams/        # Teams and roles domain
│   ├── application/      # Application layer (use cases, repositories)
│   │   ├── accounts/
│   │   ├── anime/
│   │   ├── media/
│   │   └── teams/
│   ├── infrastructure/   # Infrastructure layer (database, cache, storage)
│   │   ├── database/
│   │   ├── cache/
│   │   └── media_storage/
│   └── api/             # API layer (FastAPI routes, schemas)
│       ├── v1/
│       └── schemas/
├── config/              # Configuration management
├── migrations/          # Alembic database migrations
├── tests/              # Test suite
├── docker-compose.yml  # Docker services
├── Dockerfile          # Application container
└── pyproject.toml      # Dependencies and project metadata
```

## Architecture

This project follows **Clean Architecture** + **Hexagonal Architecture** + **DDD-lite** principles:

### Domain Layer
- Pure business logic
- Entities, Value Objects, Domain Events
- Domain Services
- No external dependencies

### Application Layer
- Use Cases (orchestration)
- Repository interfaces
- Unit of Work pattern
- Event Bus

### Infrastructure Layer
- SQLAlchemy 2.0 async models
- Redis caching
- Media storage and CDN
- External services

### API Layer
- FastAPI routes
- Pydantic v2 schemas
- Authentication & authorization
- API versioning

## Features

- **User Management**: Registration, authentication, sessions
- **Anime Catalog**: Releases, episodes, genres, franchises
- **Media Streaming**: Video player, adaptive streaming, DRM support
- **User Features**: Favorites, collections, watch history, ratings
- **Recommendations**: AI-powered content recommendations
- **Admin Panel**: Content management, user moderation
- **Parser**: Anime content parser (Kodik/DLE analog)
- **Teams & Roles**: RBAC system for team collaboration

## Tech Stack

- **Framework**: FastAPI
- **Database**: PostgreSQL with async SQLAlchemy 2.0
- **Cache**: Redis
- **Task Queue**: Celery
- **Migrations**: Alembic
- **Validation**: Pydantic v2
- **Authentication**: JWT tokens

## Getting Started

### Prerequisites

- Python 3.11+
- PostgreSQL 15+
- Redis 7+
- Poetry (for dependency management)

### Installation

1. Clone the repository:
```bash
git clone https://github.com/Montihx/backend-ani.git
cd backend-ani
```

2. Install dependencies with Poetry:
```bash
poetry install
```

3. Copy environment variables:
```bash
cp .env.example .env
```

4. Configure `.env` with your settings

### Running with Docker

```bash
docker-compose up -d
```

This will start:
- PostgreSQL on port 5432
- Redis on port 6379
- Backend API on port 8000
- Celery worker

### Running Locally

1. Start PostgreSQL and Redis

2. Run database migrations:
```bash
poetry run alembic upgrade head
```

3. Start the application:
```bash
poetry run python main.py
```

The API will be available at `http://localhost:8000`

### API Documentation

- Swagger UI: `http://localhost:8000/api/docs`
- ReDoc: `http://localhost:8000/api/redoc`

## Development

### Running Tests

```bash
poetry run pytest
```

### Code Formatting

```bash
poetry run black src/
poetry run ruff check src/
```

### Type Checking

```bash
poetry run mypy src/
```

### Creating Migrations

```bash
poetry run alembic revision --autogenerate -m "description"
poetry run alembic upgrade head
```

## Project Principles

1. **Strict Layer Separation**: No circular dependencies
2. **Domain-Driven Design**: Business logic in domain layer
3. **Dependency Inversion**: Interfaces for external dependencies
4. **Async-First**: All I/O operations are asynchronous
5. **Event-Driven**: Domain events for cross-domain communication
6. **RBAC**: Role-based access control for all features

## API Endpoints

### Authentication
- `POST /api/v1/auth/register` - Register new user
- `POST /api/v1/auth/login` - Login user
- `POST /api/v1/auth/logout` - Logout user
- `GET /api/v1/auth/me` - Get current user

### Releases
- `GET /api/v1/releases` - List releases
- `POST /api/v1/releases` - Create release
- `GET /api/v1/releases/{id}` - Get release
- `PUT /api/v1/releases/{id}` - Update release
- `DELETE /api/v1/releases/{id}` - Delete release
- `POST /api/v1/releases/search` - Search releases

### Episodes
- `POST /api/v1/episodes` - Create episode
- `GET /api/v1/releases/{id}/episodes` - List episodes
- `GET /api/v1/episodes/{id}` - Get episode
- `PUT /api/v1/episodes/{id}` - Update episode
- `POST /api/v1/episodes/{id}/release` - Release episode

### Favorites
- `GET /api/v1/favorites` - Get favorites
- `POST /api/v1/favorites/{release_id}` - Add to favorites
- `DELETE /api/v1/favorites/{release_id}` - Remove from favorites

## License

This project is licensed under the MIT License.

## Contributing

Contributions are welcome! Please follow the established architecture patterns and coding standards.
