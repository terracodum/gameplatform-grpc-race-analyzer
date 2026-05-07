# grpc-race-analyzer

Микросервис анализа телеметрии F1 гонок в реальном времени. Часть платформы [GamePlatform](https://github.com/terracodum/gameplatform).

**Основная тема:** gRPC inter-service communication, realtime обработка данных.

---

## Что делает

- Получает поток телеметрии от grpc-race-source через gRPC
- Анализирует данные в реальном времени: максимальная скорость, лучший круг, торможение, передачи
- Отдаёт результаты анализа на live дашборд через WebSocket
- Сохраняет итоги сессии в БД

---

## Стек

| Слой | Технология |
|---|---|
| Backend | Go, gRPC |
| Frontend | React, MUI |
| БД | PostgreSQL |
| Протокол | gRPC, protobuf |
| Realtime | WebSocket (фронт) |
| Миграции | goose |

---

## Архитектура

```
grpc-race-source
     │
     │ gRPC stream
     ▼
grpc-race-analyzer
     │
     ├── анализ фреймов телеметрии
     │       │
     │       └── max speed, best lap, throttle/brake зоны
     │
     ├── PostgreSQL (итоги сессий)
     │
     └── WebSocket ──► React фронт (live дашборд)
```

---

## Структура проекта

```
grpc-race-analyzer/
├── cmd/
│   └── main.go
├── internal/
│   ├── domain/        # сущности
│   ├── repository/    # работа с БД
│   ├── service/       # анализ телеметрии
│   ├── handler/       # HTTP + WebSocket handlers
│   └── hub/           # управление WebSocket соединениями
├── proto/
│   └── telemetry.proto  # protobuf схема (общая с source)
├── migrations/        # goose миграции
├── frontend/          # React приложение
├── Dockerfile
├── docker-compose.yml
└── .env.example
```

---

## Переменные окружения

```env
DATABASE_URL=postgres://user:password@localhost:5432/grpc_race_analyzer
GRPC_PORT=50051
CORE_URL=http://localhost:8080
PORT=8084
```

---

## Локальный запуск

```bash
cp .env.example .env
# заполни .env своими значениями

docker compose up --build
```

Сервис доступен на **http://localhost:8084**

> grpc-race-source должен быть запущен до старта analyzer

---

## API

| Метод | Путь | Описание |
|---|---|---|
| GET | /api/v1/sessions | История проанализированных сессий |
| GET | /api/v1/sessions/:id | Итоги конкретной сессии |
| GET | /ws/live | WebSocket для live дашборда |
| GET | /health | Health check |

---

## WebSocket события

```json
{ "type": "frame", "data": { "driver": "1", "speed": 312.5, "rpm": 11200, "gear": 8 } }
{ "type": "lap", "data": { "driver": "1", "lap": 12, "time": 83.456 } }
{ "type": "best_lap", "data": { "driver": "1", "time": 81.234 } }
{ "type": "session_end", "data": { "winner": "1", "total_laps": 57 } }
```

---

## Связь с платформой

После завершения сессии отправляет итог в core:

```
POST {CORE_URL}/api/v1/stats
{
  "service": "grpc-race",
  "userId": "...",
  "payload": { "session": "...", "bestLap": 81.234, "maxSpeed": 342.1 }
}
```