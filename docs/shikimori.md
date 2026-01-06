# Shikimori Provider — спецификация источника метаданных

> ⚠️ **КРИТИЧЕСКИ ВАЖНО**
> Этот документ является **канонической спецификацией Shikimori-провайдера** для AI-агента.
>
> Shikimori — **единственный авторитетный источник метаданных** в системе KITSU.
> Все остальные источники подчиняются моделям, определённым здесь.

---

## 1. Роль Shikimori в системе

Shikimori используется ИСКЛЮЧИТЕЛЬНО для:

* метаданных аниме
* жанров и тегов
* рейтингов
* статусов выхода
* связей между тайтлами

❌ Shikimori **НЕ** используется для:

* видео
* эпизодов
* потоков

---

## 2. Источники документации

Основа:

* SHIKI_API.md

Все ответы:

* валидируются
* нормализуются
* кешируются

---

## 3. Архитектурное положение

```
Domain Layer
   ↓ interface
ShikimoriProvider
   ↓ HTTP
Shikimori API
```

ShikimoriProvider:

* stateless
* изолирован
* заменяем

---

## 4. Конфигурация (ENV)

Обязательные переменные:

* `SHIKI_BASE_URL`
* `SHIKI_TIMEOUT`
* `SHIKI_RATE_LIMIT`

❌ Запрещено:

* хардкод
* обращение из frontend

---

## 5. Каноническая модель Anime

### 5.1 Anime (Domain Model)

Обязательные поля:

* id (internal)
* shikimoriId
* title
* titleOriginal
* description
* poster
* year
* status
* rating
* genres[]
* episodesTotal
* duration

---

### 5.2 Связи (Relations)

Типы связей:

* sequel
* prequel
* spin_off
* adaptation

Каждая связь:

* type
* targetShikimoriId

---

## 6. Интерфейсы провайдера

### 6.1 getAnimeById

**Input:**

* shikimori_id: int

**Output:**

* Anime | null

---

### 6.2 searchAnime

**Input:**

* query: string

**Output:**

* Anime[] (short)

---

### 6.3 getRelations

**Input:**

* shikimori_id: int

**Output:**

* Relation[]

---

## 7. Нормализация данных

* snake_case → camelCase
* HTML → plain text
* enum-значения строго ограничены

---

## 8. Кеширование

* агрессивное кеширование
* TTL ≥ 24 часа
* invalidation при ошибках

---

## 9. Rate Limit

* учитывать лимиты Shikimori
* очереди запросов
* backoff

---

## 10. Ошибки

* провайдерские ошибки не пробрасываются напрямую
* все ошибки маппятся в internal codes

---

## 11. Безопасность

* защита от SSRF
* фильтрация URL изображений
* запрет логирования ответов целиком

---

## 12. Тестирование

* mock Shikimori API
* проверка edge-cases
* стабильность моделей

---

## 13. Статус документа

* канонический
* обязателен
* используется AI-агентом как модель данных
