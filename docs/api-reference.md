# API Reference — Backend KITSU

> ⚠️ **КАНОНИЧЕСКИЙ ДОКУМЕНТ**
> Этот файл определяет **ВСЕ публичные REST API эндпоинты** backend’а KITSU.
> Используется одновременно:
>
> * Frontend
> * Backend
> * AI Copilot / Agent
>
> ❌ Добавлять эндпоинты без изменения этого файла запрещено
> ❌ Менять контракты без версии запрещено

---

## 1. Общие правила API

### 1.1 Протокол

* HTTPS only
* REST
* JSON

---

### 1.2 Версионирование

Все эндпоинты:

```
/api/v1/...
```

Изменение структуры ответа = новая версия API.

---

### 1.3 Общий формат ошибок

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human readable message"
  }
}
```

---

## 2. Аутентификация

На первом этапе:

* публичный API
* без логина

Подготовка:

* JWT
* OAuth

---

## 3. Anime

### 3.1 Получить аниме по ID

**GET** `/api/v1/anime/{id}`

**Response:**

```json
{
  "id": 1,
  "title": "Название",
  "poster": "url",
  "rating": 8.5
}
```

---

### 3.2 Поиск аниме

**GET** `/api/v1/anime/search?q=naruto`

**Response:**

```json
[
  {
    "id": 1,
    "title": "Naruto"
  }
]
```

---

## 4. Episodes

### 4.1 Список эпизодов

**GET** `/api/v1/anime/{id}/episodes`

**Response:**

```json
[
  {
    "episode": 1,
    "translations": ["Sub", "Dub"]
  }
]
```

---

## 5. Streams

### 5.1 Получить потоки

**GET** `/api/v1/episodes/{episodeId}/streams`

**Response:**

```json
[
  {
    "quality": "720p",
    "url": "m3u8_url"
  }
]
```

---

## 6. Player State

### 6.1 Сохранить прогресс

**POST** `/api/v1/player/progress`

**Body:**

```json
{
  "episodeId": "123",
  "time": 532
}
```

---

## 7. Ограничения

* rate limit
* защита от abuse

---

## 8. Статус документа

* обязателен
* source of truth
* используется Copilot
