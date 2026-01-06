# Phase 5: Dashboard, Аналитика и Подписки

## Обзор

Phase 5 реализует систему аналитики, dashboard для администраторов, и функционал подписок/уведомлений. Все компоненты разработаны с учетом работы на слабых серверах: асинхронные операции, Redis кеширование, оптимизированные запросы к БД.

---

## Структура проекта Phase 5

### API Endpoints

**Расположение:** `src/api/v1/routes/`

#### 1. Analytics (`analytics.py`)
- **GET** `/api/v1/analytics/views` - Аналитика просмотров релизов
  - Параметры: `release_ids` (опционально), `limit`
  - Кеширование: 10 минут в Redis
  - Возвращает: общее количество просмотров, уникальные зрители, средний процент завершения
  
- **GET** `/api/v1/analytics/top` - Топ релизов и жанров по популярности
  - Параметры: `limit` (1-50)
  - Кеширование: 1 час в Redis
  - Возвращает: топ релизов по индексу популярности, топ жанров по просмотрам

#### 2. Subscriptions & Notifications (`subscriptions.py`)
- **POST** `/api/v1/subscriptions` - Создать подписку
  - Типы: release, genre, franchise
  - Требует: аутентификация
  
- **GET** `/api/v1/subscriptions` - Список подписок пользователя
  - Поддержка пагинации
  
- **DELETE** `/api/v1/subscriptions/{id}` - Отписаться
  
- **GET** `/api/v1/notifications` - Список уведомлений
  - Параметры: `unread_only`, `skip`, `limit`
  
- **POST** `/api/v1/notifications/{id}/read` - Отметить как прочитанное
  
- **GET** `/api/v1/notifications/unread/count` - Количество непрочитанных

#### 3. CMS (`cms.py`)
- **POST** `/api/v1/admin/releases` - Создать релиз (только админ)
- **PUT** `/api/v1/admin/releases/{id}` - Обновить релиз (только админ)
- **DELETE** `/api/v1/admin/releases/{id}` - Удалить релиз (только админ)
- **POST** `/api/v1/admin/episodes` - Создать эпизод (только админ)
- **PUT** `/api/v1/admin/episodes/{id}` - Обновить эпизод (только админ)
- **DELETE** `/api/v1/admin/episodes/{id}` - Удалить эпизод (только админ)

---

### Pydantic Схемы

**Расположение:** `src/api/schemas/`

#### 1. `analytics.py`
- **ViewsAnalyticsRequest** - Запрос аналитики просмотров
- **ReleaseViewsItem** - Метрики просмотров одного релиза
- **ViewsAnalyticsResponse** - Ответ с аналитикой просмотров
- **TopReleaseItem** - Элемент топового релиза
- **TopGenreItem** - Элемент топового жанра
- **TopAnalyticsResponse** - Ответ с топами релизов и жанров
- **DashboardStats** - Общая статистика для dashboard
- **DashboardResponse** - Полный ответ dashboard

#### 2. `subscriptions.py`
- **SubscriptionCreateRequest** - Запрос создания подписки
- **SubscriptionResponse** - Ответ с данными подписки
- **SubscriptionListResponse** - Список подписок
- **NotificationResponse** - Данные уведомления
- **NotificationListResponse** - Список уведомлений
- **NotificationMarkReadRequest** - Отметить уведомление

#### 3. `dashboard.py`
- **ReleaseCreateRequest** - Создание релиза (CMS)
- **ReleaseUpdateRequest** - Обновление релиза (CMS)
- **ReleaseAdminResponse** - Ответ CMS для релиза
- **EpisodeCreateRequest** - Создание эпизода (CMS)
- **EpisodeUpdateRequest** - Обновление эпизода (CMS)
- **EpisodeAdminResponse** - Ответ CMS для эпизода

---

### Модели БД

**Расположение:** `src/infrastructure/database/models/anime.py`

#### 1. ViewMetricsModel
```python
- id: UUID (PK)
- release_id: UUID (индекс)
- episode_id: UUID (опционально, индекс)
- user_id: UUID (опционально, индекс)
- session_id: String (для анонимных)
- duration_seconds: Integer
- completion_percentage: Float (0-100)
- viewed_at: DateTime (индекс)
```
**Индексы:**
- `release_id` - для быстрого получения просмотров релиза
- `episode_id` - для аналитики эпизодов
- `user_id` - для уникальных зрителей
- `viewed_at` - для временных фильтров

#### 2. PopularityMetricsModel
```python
- id: UUID (PK)
- release_id: UUID (уникальный индекс)
- total_views: Integer
- unique_viewers: Integer
- average_rating: Float
- popularity_score: Float (индекс для сортировки)
- last_calculated_at: DateTime
- updated_at: DateTime
```
**Индексы:**
- `release_id` - уникальный для upsert операций
- `popularity_score` - для быстрой сортировки топов

#### 3. SubscriptionModel
```python
- id: UUID (PK)
- user_id: UUID (индекс)
- subscription_type: String (release/genre/franchise)
- target_id: UUID (индекс)
- created_at: DateTime
- is_active: Boolean
```
**Индексы:**
- `user_id` - для получения подписок пользователя
- `target_id` - для поиска подписчиков объекта
- Композитный индекс `(subscription_type, target_id, is_active)` - для эффективного поиска подписчиков

**Ограничения:**
- Уникальный индекс на `(user_id, subscription_type, target_id)` - предотвращение дублей

#### 4. NotificationModel
```python
- id: UUID (PK)
- user_id: UUID (индекс)
- notification_type: String
- title: String
- message: Text
- related_id: UUID (опционально)
- is_read: Boolean
- created_at: DateTime (индекс)
- read_at: DateTime (опционально)
```
**Индексы:**
- `user_id` - для получения уведомлений пользователя
- `created_at` - для сортировки по дате
- Композитный индекс `(user_id, is_read)` - для быстрого получения непрочитанных

---

### Репозитории

**Расположение:** `src/infrastructure/database/repositories/`

#### 1. `analytics.py`

**SQLAlchemyViewMetricsRepository**
- `create()` - Записать событие просмотра
- `get_views_by_release()` - Общее количество просмотров релиза
- `get_views_by_episode()` - Просмотры эпизода
- `get_unique_viewers_by_release()` - Уникальные зрители релиза
- `get_average_completion_by_release()` - Средний процент завершения

**SQLAlchemyPopularityMetricsRepository**
- `get_by_release_id()` - Получить метрики релиза
- `upsert()` - Создать/обновить метрики (для фоновых задач)
- `get_top_by_views()` - Топ по просмотрам
- `get_top_by_popularity_score()` - Топ по индексу популярности

#### 2. `dashboard.py`

**SQLAlchemySubscriptionRepository**
- `create()` - Создать подписку
- `get_by_id()` - Получить по ID
- `get_by_user()` - Все подписки пользователя
- `exists()` - Проверить существование
- `delete()` - Деактивировать (soft delete)
- `get_subscribers_by_target()` - Все подписчики объекта

**SQLAlchemyNotificationRepository**
- `create()` - Создать уведомление
- `get_by_user()` - Уведомления пользователя
- `mark_as_read()` - Отметить как прочитанное
- `count_unread()` - Количество непрочитанных
- `count_recent_notifications()` - Для rate limiting
- `delete_old_read_notifications()` - Очистка старых (фоновая задача)

---

## Кеширование (Redis)

### Стратегия кеширования

1. **Аналитика просмотров** - TTL 10 минут
   - Ключ: `analytics:views:{release_ids}:{limit}`
   - Обновление: автоматическое по истечению TTL
   - Fallback: прямой запрос к БД при недоступности Redis

2. **Топ релизов и жанров** - TTL 1 час
   - Ключ: `analytics:top:{limit}`
   - Обновление: автоматическое по истечению TTL
   - Расчет тяжелый, поэтому долгий TTL

3. **Подписки и уведомления** - не кешируются
   - Причина: данные должны быть актуальными в реальном времени
   - Оптимизация: эффективные индексы БД

### Реализация в коде

```python
# Попытка получить из кеша
try:
    cache = RedisCache(settings.redis_url) if settings.redis_url else None
    if cache:
        cached = await cache.get(cache_key)
        if cached:
            return Response.model_validate_json(cached)
except Exception as e:
    logger.warning(f"Ошибка чтения кеша: {e}")

# Запрос к БД
async with uow:
    # ... получение данных ...
    
# Сохранить в кеш
try:
    if cache:
        await cache.set(cache_key, response.model_dump_json(), ttl=600)
except Exception as e:
    logger.warning(f"Ошибка записи в кеш: {e}")
```

---

## Rate Limiting

### Middleware уровень
- **RateLimitMiddleware** (`src/api/middleware/rate_limit.py`)
- Лимиты по умолчанию: 60 запросов/минуту на IP
- Исключения: `/health`, `/api/docs`

### Уровень приложения
- **Уведомления**: максимум N уведомлений в час для одного пользователя
  - Проверка в `count_recent_notifications()`
  - Предотвращение спама

### Конфигурация
```python
# config/settings.py
rate_limit_enabled: bool = True
rate_limit_requests_per_minute: int = 60
```

---

## Асинхронные операции

### Все операции async
Все функции репозиториев и endpoints используют `async/await`:
- Минимальная блокировка I/O
- Эффективное использование ресурсов сервера
- Поддержка множественных одновременных запросов

### Примеры
```python
async def get_views_analytics(...) -> ViewsAnalyticsResponse:
    async with uow:
        # Все операции БД асинхронные
        total_views = await view_metrics_repo.get_views_by_release(release_id)
        unique_viewers = await view_metrics_repo.get_unique_viewers_by_release(release_id)
```

### Celery задачи для тяжелых операций
- Обновление метрик популярности - фоновая задача
- Рассылка уведомлений подписчикам - фоновая задача
- Очистка старых уведомлений - периодическая задача

Расположение: `src/infrastructure/tasks/notifications.py`

---

## Оптимизация для слабых серверов

### 1. Предварительный расчет метрик
- PopularityMetricsModel хранит pre-calculated данные
- Обновление через Celery в фоне
- Быстрое чтение без сложных агрегаций

### 2. Эффективные индексы БД
- Все часто используемые поля индексированы
- Композитные индексы для сложных запросов
- Минимизация full table scans

### 3. Redis кеширование
- Снижение нагрузки на PostgreSQL
- Быстрые ответы для популярных запросов
- Graceful fallback при недоступности

### 4. Пагинация везде
- Лимиты по умолчанию: 50 записей
- Максимум: 100 записей за запрос
- Защита от перегрузки памяти

### 5. Мягкое удаление (Soft Delete)
- Подписки: `is_active = False`
- Сохранение истории
- Меньше write операций

---

## Безопасность

### Аутентификация
- Все endpoints подписок/уведомлений требуют JWT токен
- Проверка через `get_current_user_id()`

### Авторизация
- CMS endpoints требуют роль ADMIN
- Проверка через `require_role(UserRole.ADMIN)`

### Проверка владельца
- Пользователи могут управлять только своими подписками
- Пользователи видят только свои уведомления
- Защита от несанкционированного доступа

```python
if subscription.user_id != user_id:
    raise HTTPException(status_code=403, detail="Нельзя удалить чужую подписку")
```

---

## Миграции БД

**Файл:** `alembic/versions/003_phase5_analytics_subscriptions.py`

Создает все таблицы Phase 5:
- `view_metrics` - с индексами на release_id, episode_id, user_id, viewed_at
- `popularity_metrics` - с уникальным индексом на release_id
- `subscriptions` - с индексами и уникальным ограничением
- `notifications` - с индексами на user_id, is_read, created_at

**Запуск миграции:**
```bash
alembic upgrade head
```

---

## Тестирование

**Расположение:** `tests/integration/test_phase5.py`

### Покрытые сценарии
1. Аналитика - получение просмотров
2. Аналитика - топ релизов и жанров
3. Подписки - создание, получение, удаление
4. Уведомления - создание, получение, отметка как прочитанное
5. CMS - требование авторизации для админ endpoints

### Запуск тестов
```bash
pytest tests/integration/test_phase5.py -v
```

---

## Интеграция с другими Phase

### Phase 1-3: Основа
- Использует модели User, Release, Episode
- Интегрируется с системой аутентификации
- Использует общие репозитории

### Phase 4: Расширенные функции
- Дополняет профиль пользователя метриками
- Интегрируется с рекомендациями
- Добавляет контекст для комментариев

### Phase 6: Будущее (в разработке)
- Расширенная аналитика
- ML-powered рекомендации на основе метрик
- Персонализированные уведомления

---

## API Documentation

Полная документация доступна по адресу:
- **Swagger UI:** `http://localhost:8000/api/docs`
- **ReDoc:** `http://localhost:8000/api/redoc`

---

## Русские комментарии

Все файлы Phase 5 содержат подробные комментарии на русском языке:
- Описание классов и функций
- Пояснения по бизнес-логике
- Примечания по оптимизации
- Предупреждения о безопасности

---

## Заключение

Phase 5 предоставляет полноценную систему аналитики, управления контентом и уведомлений, оптимизированную для работы на слабых серверах. Архитектура чистая, код читаемый, документация полная.

**Статус:** ✅ Полностью реализован и готов к интеграции с Phase 6
