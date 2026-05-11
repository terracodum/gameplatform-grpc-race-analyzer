# grpc-race

Часть платформы [GamePlatform](https://github.com/terracodum/gameplatform).

Система из двух сервисов которые общаются через gRPC. Основная цель — изучение gRPC streaming и inter-service communication на Go.

---

## Сервисы

### grpc-race-source
- Тянет исторические данные телеметрии F1 с OpenF1 API
- Симулирует live воспроизведение с реальными таймингами
- Стримит телеметрию в grpc-race-analyzer через gRPC server-side streaming
- HTTP API для управления сессиями (старт/стоп)

### grpc-race-analyzer
- Получает поток телеметрии от source через gRPC
- Анализирует данные: макс скорость, лучший круг, throttle/brake зоны
- Отдаёт результаты на фронт через WebSocket
- Сохраняет итоги сессий в БД

---

## Стек

- **Backend:** Go, chi
- **Протокол:** gRPC, protobuf
- **БД:** PostgreSQL
- **Миграции:** goose
- **Realtime (фронт):** WebSocket
- **Frontend:** React, MUI
- **API:** [OpenF1](https://openf1.org) — бесплатный, исторические данные F1

---

## Архитектура

```
React фронт
     │
     │ HTTP/JSON
     ▼
grpc-race-source (port 8083, gRPC port 50050)
     │
     ├── OpenF1 API
     │
     └── gRPC stream ──► grpc-race-analyzer (port 8084, gRPC port 50051)
                                   │
                               WebSocket ──► React фронт (live дашборд)
                                   │
                               PostgreSQL
```

---

## Proto схема

```protobuf
syntax = "proto3";

message TelemetryFrame {
  string driver_number = 1;
  float  speed         = 2;
  float  rpm           = 3;
  float  throttle      = 4;
  float  brake         = 5;
  int32  gear          = 6;
  float  lap_time      = 7;
  int32  lap_number    = 8;
  int64  timestamp     = 9;
}

message StreamRequest {
  string session_key   = 1;
  string driver_number = 2;
}

service TelemetryService {
  rpc StreamTelemetry(StreamRequest) returns (stream TelemetryFrame);
}
```

---

## Переменные окружения

### grpc-race-source
```env
DATABASE_URL=postgres://user:password@localhost:5432/grpc_race_source
ANALYZER_GRPC_URL=localhost:50051
CORE_URL=http://localhost:8080
PORT=8083
GRPC_PORT=50050
```

### grpc-race-analyzer
```env
DATABASE_URL=postgres://user:password@localhost:5432/grpc_race_analyzer
GRPC_PORT=50051
CORE_URL=http://localhost:8080
PORT=8084
```

---

## Локальный запуск

```bash
# терминал 1 — сначала analyzer
cd grpc-race-analyzer
go run cmd/main.go

# терминал 2
cd grpc-race-source
go run cmd/main.go
```

---

## Структура проекта (одинакова для обоих)

```
grpc-race-*/
├── cmd/
│   └── main.go
├── internal/
│   ├── domain/
│   ├── repository/
│   ├── service/
│   └── handler/
├── proto/
│   └── telemetry.proto
├── migrations/
├── frontend/
├── Dockerfile
├── docker-compose.yml
└── .env.example
```

---

## Связь с платформой

После завершения сессии analyzer отправляет итог в core:

```
POST {CORE_URL}/api/v1/stats
{
  "service": "grpc-race",
  "userId": "...",
  "payload": { "session": "...", "bestLap": 81.234, "maxSpeed": 342.1 }
}
```

---

## OpenF1 API

Документация: https://openf1.org

Полезные эндпоинты:
- `/v1/sessions` — список сессий (гонки, квалификации)
- `/v1/car_data` — телеметрия: speed, rpm, throttle, brake, gear
- `/v1/laps` — время кругов
- `/v1/drivers` — информация о пилотах

Лимит: 30 запросов в минуту. Кэшируй сессии локально.