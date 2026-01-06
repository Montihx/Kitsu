# Kodik Provider — спецификация парсера

> ⚠️ **КРИТИЧЕСКИ ВАЖНО**
> Данный файл является **строгой спецификацией для AI-агента (Copilot)**.
>
> ❌ Запрещено использовать iframe
> ❌ Запрещено проксировать видео
> ❌ Запрещено отдавать API Kodik напрямую во frontend
>
> Любое нарушение = критическая архитектурная ошибка.

---

## 1. Роль Kodik в системе

**Kodik** используется ИСКЛЮЧИТЕЛЬНО как:

* источник эпизодов
* источник переводов
* источник HLS (.m3u8)

Kodik **НЕ** используется:

* как UI
* как iframe-плеер
* как CDN

---

## 2. Источник документации

Основа:

* KODIK_API.md

Все данные должны:

* валидироваться
* нормализоваться
* кешироваться

---

## 3. Архитектурное положение

```
Domain Layer
   ↓ interface
KodikProvider
   ↓ HTTP
Kodik API
```

KodikProvider:

* изолирован
* заменяем
* stateless

---

## 4. Конфигурация (ENV)

Обязательные переменные:

* `KODIK_API_KEY`
* `KODIK_BASE_URL`
* `KODIK_TIMEOUT`

❌ Запрещено:

* хардкод
* дефолтные ключи

---

## 5. Интерфейс провайдера

### 5.1 getAnimeByShikimoriId

**Input:**

* shikimori_id: int (required)

**Output:**

* KodikAnime | null

**Errors:**

* PROVIDER_UNAVAILABLE
* INVALID_RESPONSE

---

### 5.2 getEpisodes

**Input:**

* kodik_id: string

**Output:**

* Episode[]

Episode:

* episode_number: int
* title: string | null
* translations: Translation[]

---

### 5.3 getStreams

**Input:**

* episode_id: string

**Output:**

* Stream[]

Stream:

* quality: string (360p, 480p, 720p, 1080p)
* url: string (.m3u8)

❌ Прямые .mp4 запрещены

---

## 6. Нормализация данных

* привести названия полей к camelCase
* удалить HTML
* привести типы

---

## 7. HLS требования

* проверка валидности m3u8
* fallback при ошибках
* таймауты

---

## 8. Rate Limit и Retry

* учитывать лимиты Kodik
* exponential backoff
* max retry = 3

---

## 9. Кеширование

* кешировать ответы
* TTL configurable
* ключ = shikimori_id

---

## 10. Ошибки

* ошибки провайдера не пробрасываются напрямую
* всегда маппинг на internal error codes

---

## 11. Безопасность

* защита от SSRF
* валидация URL
* запрет логирования токенов

---

## 12. Тестирование

* unit tests
* mock Kodik API
* edge-cases

---

## 13. Статус документа

* обязателен к соблюдению
* используется Copilot как контракт
