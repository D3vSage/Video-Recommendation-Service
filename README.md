
# Video Recommendation Service

Асинхронный сервис рекомендаций видео на основе **жанров** и **пользовательских взаимодействий**.

Система сочетает:

* content-based рекомендации (по жанрам),
* user-item матрицу взаимодействий,
* асинхронную обработку событий через RabbitMQ,
* REST API на FastAPI.

---

## Основная идея

Рекомендации формируются на основе:

1. **Схожести видео по жанрам** (cosine similarity).
2. **Истории взаимодействий пользователей** (view, like, comment, favorite).
3. Взвешивания действий и нормализации итоговых скорингов.

Система обновляется **в реальном времени** при поступлении новых событий из RabbitMQ.

---

## Архитектура проекта

```
.
├── domain/
│   ├── entities.py        # Сущности
│   └── use_cases.py       # Бизнес-логика
│
├── infrastructure/
│   ├── db.py              # PostgreSQL
│   └── rabbitmq.py        # RabbitMQ
│
├── interfaces/
│   ├── api.py             # FastAPI REST API
│   └── consumer.py        # RabbitMQ
│
├── main.py
├── send_interactions.py   # Генератор тестовых событий
└── config.py              # Конфигурация БД
```

---

## Используемые технологии

* **Python 3.10+**
* **FastAPI**
* **PostgreSQL** + `asyncpg`
* **RabbitMQ** + `aio-pika`
* **Pandas / NumPy**
* **scikit-learn**
* **Uvicorn**

---

## Логика рекомендаций

### 1. Схожесть видео

* Жанры кодируются с помощью `MultiLabelBinarizer`
* Вычисляется `cosine_similarity`
* Матрица сохраняется в БД (`video_similarity`)

### 2. User–Item матрица

* Строки — пользователи
* Столбцы — видео
* Значения — взвешенные действия:

```python
action_weights = {
    "view": 1.0,
    "like": 2.0,
    "comment": 3.0,
    "favorite": 4.0
}
```

### 3. Рекомендации

* `video_similarity × user_ratings`
* Нормализация скорингов
* Исключение уже просмотренных видео
* Fallback на популярные видео при отсутствии данных

---

## Запуск проекта

### Подготовка инфраструктуры

Убедитесь, что запущены:

* PostgreSQL
* RabbitMQ

Пример таблиц:

```sql
CREATE TABLE videos (
    id TEXT PRIMARY KEY,
    genres TEXT[]
);

CREATE TABLE interactions (
    user_id TEXT,
    video_id TEXT,
    action TEXT,
    UNIQUE (user_id, video_id, action)
);

CREATE TABLE video_similarity (
    video1_id TEXT,
    video2_id TEXT,
    similarity FLOAT
);
```

---

### Установка зависимостей

```bash
pip install -r requirements.txt
```

---

### Конфигурация

Файл `config.py`:

```python
host = "127.0.0.1"
user = "postgres"
password = "qwerty"
db_name = "recommendations"
```

---

### Запуск сервиса

```bash
python main.py
```

Сервис будет доступен по адресу:

```
http://localhost:8000
```

---

## REST API

### Получить рекомендации

```
GET /recommendations/{user_id}
```

#### Пример ответа:

```json
[
  { "video_id": "video_42", "score": 0.91 },
  { "video_id": "video_17", "score": 0.84 },
  { "video_id": "video_8",  "score": 0.79 }
]
```

---

## RabbitMQ

* Очередь: `interactions_queue`
* Формат сообщения:

```json
{
  "user_id": "user_12",
  "video_id": "video_5",
  "action": "like"
}
```

Consumer:

* сохраняет событие в БД,
* обновляет user-item матрицу,
* пересчитывает рекомендации для пользователя.

---

## Генерация тестовых данных

Скрипт для нагрузки и тестирования:

```bash
python send_interactions.py
```

Он:

* читает `video_id` из БД,
* генерирует случайные действия пользователей,
* отправляет их в RabbitMQ.

---

## Возможные улучшения

* Гибридная модель (content + collaborative filtering)
* Time-decay для взаимодействий
* Кеширование рекомендаций (Redis)
* Batch-переобучение по расписанию
* ML-модель вместо линейного скоринга
