# Студийный рекордер

Студийный рекордер - система для записи, контроля и постобработки RTSP-аудио и RTSP-видео по аудиториям/комнатам.

Система состоит из нескольких сервисов:

- **agent** - записывает аудио- и видеопотоки конкретной комнаты;
- **master** - управляет пулом агентов, комнатами и запуском постобработки;
- **frontend** - веб-интерфейс для работы с master;
- **algorithm** - CLI-инструмент `postprocess-room` для сборки итоговых файлов;
- **postgres** - база конфигурации комнат, потоков и агентов;
- **prometheus** и **grafana** - мониторинг, алерты и дашборды.

Этот README описывает проект целиком: архитектуру, запуск через Docker Compose, порты, мониторинг и важные эксплуатационные детали. Подробности конкретных сервисов лежат в их README:

- [recorder/agent/README.MD](recorder/agent/README.MD)
- [recorder/master/README.md](recorder/master/README.md)
- [recorder/algorithm/README.md](recorder/algorithm/README.md)

---

## Быстрый запуск

Требования:

- Docker и Docker Compose;
- доступ к RTSP-камерам/микрофонам из контейнеров;
- свободные порты `3000`, `5173`, `5433`, `8082`, `8083`, `8090`, `9090`;
- каталог `videos` для записей.

Запуск из корня проекта:

```bash
mkdir -p videos
docker compose up --build -d
```

Проверка:

```bash
docker compose ps
curl http://localhost:8090/health
curl http://localhost:8082/health
curl http://localhost:8083/health
```

Остановка:

```bash
docker compose down
```

Остановка с удалением данных PostgreSQL, Prometheus и Grafana:

```bash
docker compose down -v
```

Если контейнеры agent/master получают `permission denied` при записи файлов, проверьте права на каталог:

```bash
sudo chown -R 1000:1000 ./videos
```

---

## Адреса сервисов

| Сервис | URL на хосте | Назначение |
| --- | --- | --- |
| Frontend | `http://localhost:5173` | Веб-интерфейс управления master |
| Master API | `http://localhost:8090` | Центральный API записи |
| Master Swagger | `http://localhost:8090/swagger/index.html` | Документация API master |
| Agent 1 | `http://localhost:8082` | Первый recorder agent |
| Agent 2 | `http://localhost:8083` | Второй recorder agent |
| PostgreSQL | `localhost:5433` | База `recorder` |
| Prometheus | `http://localhost:9090` | Метрики и правила алертов |
| Grafana | `http://localhost:3000` | Дашборды, логин `admin`, пароль `admin` |

Внутри Docker-сети сервисы обращаются друг к другу по именам из `docker-compose.yaml`: `master`, `agent-1`, `agent-2`, `postgres`, `prometheus`, `grafana`.

---

## Состав системы

### Agent

Agent отвечает за непосредственную запись потоков.

Что делает agent:

- принимает конфигурацию комнаты через `POST /config`;
- проверяет RTSP URL и состав потоков;
- инициализирует аудио- и видеозапись через GStreamer;
- запускает и останавливает запись;
- сохраняет исходные сегменты и status-файлы;
- отдаёт состояние потоков, комнаты и метрики Prometheus.

Обычно один agent обслуживает одну комнату / один набор потоков в один момент времени.

### Master

Master - управляющий сервис и основная точка входа для пользователя, UI и интеграций.

Что делает master:

- читает список агентов из PostgreSQL;
- читает комнаты и RTSP-потоки из PostgreSQL;
- принимает команды от frontend, пользователя или Bitfocus Companion;
- выбирает свободного agent;
- отправляет agent конфигурацию комнаты;
- запускает и останавливает запись;
- собирает статусы по комнатам;
- после остановки запускает `postprocess-room` для каждого video-потока.

Master нужен для того, чтобы не обращаться к каждому agent напрямую.

### Algorithm

Algorithm - сервис/CLI постобработки.

Что делает `postprocess-room`:

- читает результаты записи, полученные от agent;
- анализирует status-файлы и временные промежутки;
- склеивает видеосегменты;
- вставляет чёрные видео-заглушки на разрывах;
- склеивает аудиосегменты;
- вставляет тишину на разрывах связи;
- делает mux `video_joined + master_audio`;
- формирует итоговые файлы `output/<name>__final.mkv`.

Algorithm работает после записи или на уже готовом наборе файлов.

---

## Как сервисы связаны

```text
Пользователь / Frontend / Bitfocus
        |
        v
      Master
        |
        v
      Agent
        |
        v
  Исходные записи и status-файлы
        |
        v
 postprocess-room / Algorithm
        |
        v
Итоговые обработанные файлы
```

Сценарий start/stop:

1. Пользователь вызывает `POST /toggle` на master или нажимает кнопку в UI.
2. Master ищет комнату в PostgreSQL.
3. Master выбирает свободного agent.
4. Master отправляет agent `POST /config`.
5. Master отправляет agent `POST /start`.
6. При повторном `POST /toggle` master отправляет agent `POST /stop`.
7. Master запускает postprocess в фоне.

---

## Конфигурация данных

PostgreSQL инициализируется файлом [recorder/master/backend/db/init.sql](recorder/master/backend/db/init.sql):

- `agents` - список доступных agent-сервисов;
- `rooms` - список комнат;
- `streams` - RTSP-потоки комнат.

По умолчанию `init.sql` добавляет два agent:

```sql
('agent-1', 'http://agent-1:8082'),
('agent-2', 'http://agent-2:8083')
```

Комнаты и потоки можно добавить вручную через PostgreSQL или импортировать из CSV утилитой `cmd/import-cameras`; подробности находятся в [README master](recorder/master/README.md).

Подключение к базе с хоста:

```bash
docker exec -it recorder-postgres psql -U recorder -d recorder
```

Пример ручного добавления комнаты:

```sql
INSERT INTO rooms (room_number) VALUES ('303')
ON CONFLICT DO NOTHING;

INSERT INTO streams (room_number, name, type, url) VALUES
  ('303', 'V1', 'video', 'rtsp://192.168.1.10/stream1'),
  ('303', 'Audio1', 'audio', 'rtsp://192.168.1.20/stream0')
ON CONFLICT (room_number, name) DO UPDATE
SET type = EXCLUDED.type, url = EXCLUDED.url;
```

---

## Импорт конфигурации из CSV

Для массовой загрузки комнат и RTSP-потоков используется утилита master backend:

```text
recorder/master/backend/cmd/import-cameras
```

Она читает CSV и:

- добавляет комнаты в таблицу `rooms`;
- добавляет или обновляет потоки в таблице `streams`;
- не меняет таблицу `agents`.

Обязательные колонки CSV:

```text
Помещение
RTSP_1
RTSP_2
Звук
```

Правила импорта:

- для URL используется `RTSP_1`, а если он пустой - `RTSP_2`;
- для каждой строки создаётся video-поток;
- если `Звук` равен `да`, `yes`, `true`, `1` или `+`, дополнительно создаётся audio-поток с тем же URL;
- имена потоков генерируются автоматически: `V1`, `V2`, ... и `Audio1`, `Audio2`, ... внутри каждой комнаты;
- существующие потоки обновляются по ключу `(room_number, name)`.

Пример запуска при поднятом compose и PostgreSQL на `localhost:5433`:

```bash
cd recorder/master/backend
DATABASE_URL="postgres://recorder:recorder@localhost:5433/recorder?sslmode=disable" \
go run ./cmd/import-cameras cameras.csv
```

После импорта можно проверить результат:

```bash
curl http://localhost:8090/rooms
```

Если нужно переимпортировать комнаты с нуля, сначала очистите таблицы комнат и потоков:

```bash
docker exec -it recorder-postgres psql -U recorder -d recorder \
  -c "DELETE FROM streams; DELETE FROM rooms;"
```

---

## Запись комнаты

Через master:

```bash
curl -X POST http://localhost:8090/toggle \
  -H "Content-Type: application/json" \
  -d '{"room_id":"303"}'
```

Повторный вызов для той же комнаты остановит запись:

```bash
curl -X POST http://localhost:8090/toggle \
  -H "Content-Type: application/json" \
  -d '{"room_id":"303"}'
```

Статус системы:

```bash
curl http://localhost:8090/status
curl http://localhost:8090/rooms
curl http://localhost:8090/rooms/303/status
```

---

## Файлы записи

При запуске через Docker Compose записи доступны на хосте в:

```text
./videos
```

Master передаёт agent путь назначения через конфигурацию комнаты. В compose используется:

```yaml
DESTINATION: /app/videos
```

Пример структуры записи:

```text
videos/
  303/
    V1_videoRTSP/
      V1_videoRTSP_1.mkv
    V2_videoRTSP/
      V2_videoRTSP_1.mkv
    Audio1_audioRTSP/
      Audio1_audioRTSP_1.mkv
    V1__videoRTSP__status.json
    Audio1__audioRTSP__status.json
    output/
      V1__final.mkv
      V2__final.mkv
```

---

## Мониторинг

В `docker-compose.yaml` добавлены Prometheus и Grafana.

Prometheus доступен на:

```text
http://localhost:9090
```

Grafana доступна на:

```text
http://localhost:3000
```

Prometheus забирает метрики с:

- `master:8090/metrics`;
- `agent-1:8082/metrics`;
- `agent-2:8083/metrics`.

Правила алертов лежат в [prometheus/alerts/recorder.yml](prometheus/alerts/recorder.yml). Дашборды Grafana лежат в [grafana/provisioning/dashboards](grafana/provisioning/dashboards).

### Метрики master

Master отдаёт агрегированное состояние пула агентов и postprocess-задач:

| Метрика | Тип | Описание |
| --- | --- | --- |
| `master_agents_total` | Gauge | Общее количество агентов, загруженных мастером. |
| `master_agents_healthy` | Gauge | Количество агентов, которые master сейчас считает доступными. |
| `master_agents_busy` | Gauge | Количество агентов, которые master сейчас считает занятыми записью. |
| `master_active_recordings` | Gauge | Количество активных записей, которые master держит в памяти. |
| `master_postprocess_runs_total{result}` | Counter | Количество обработанных postprocess video-потоков с результатом `success` или `failed`. |
| `master_postprocess_duration_seconds` | Histogram | Длительность выполнения postprocess одного video-потока. |
| `master_postprocess_active` | Gauge | Количество postprocess-задач, выполняющихся прямо сейчас. |

### Метрики agent

Agent отдаёт состояние потоков и комнаты:

| Метрика | Тип | Labels | Описание |
| --- | --- | --- | --- |
| `record_state` | Gauge | `name`, `url`, `state`, `error` | Состояние отдельного потока записи. |
| `room_state` | Gauge | `Room`, `state` | Общее состояние записи комнаты. |

Значения `record_state`:

| Значение | Смысл |
| --- | --- |
| `1` | Поток работает. |
| `0` | Поток остановлен или находится в нейтральном состоянии. |
| `-1` | Ошибка незначимого потока. |
| `-2` | Ошибка значимого потока. |

Значения `room_state`:

| Значение | Смысл |
| --- | --- |
| `1` | Идёт запись. |
| `0.5` | Запись останавливается. |
| `0` | Запись остановлена. |
| `-1` | Нет потоков на запись. |
| `-2` | Упал значимый поток. |
| `-3` | Упали все потоки. |

---

## Логи

В `agent` и `master` поддерживается переменная окружения `LOG_LEVEL`:

```text
debug
info
warn
error
```

По умолчанию в `docker-compose.yaml` используется:

```text
LOG_LEVEL=info
```

При `info` не выводятся технические access logs, CORS logs, штатные RTSP-события, первый буфер потока, EOS и подробные команды postprocess. Ошибки записи, проблемы pipeline, ошибки postprocess и важные события жизненного цикла остаются в логах.

Для подробной диагностики можно временно включить:

```yaml
LOG_LEVEL: debug
```

Запуск через `docker compose up -d` не выводит логи в текущий терминал, но Docker всё равно собирает их и хранит. Смотреть их можно так:

```bash
docker compose logs -f master
docker compose logs -f agent-1
docker compose logs -f agent-2
docker compose logs -f frontend
```

---

## Когда использовать каждый сервис

Использовать master, когда нужно:

- управлять системой из одной точки;
- получать общий статус;
- работать через UI, Bitfocus или единый API;
- автоматически запускать postprocess после остановки записи.

Использовать agent напрямую, когда нужно:

- записывать конкретную комнату без master;
- диагностировать проблемы записи;
- смотреть низкоуровневое состояние потоков;
- проверять конфигурацию RTSP-потоков.

Использовать algorithm напрямую, когда нужно:

- пересобрать итоговые файлы из уже записанных сегментов;
- проверить план таймлайна;
- отладить склейку видео/аудио;
- запустить постобработку вне master.

---

## Структура проекта

```text
recorder/
  agent/              # Recorder Agent: RTSP запись, status, metrics
  algorithm/          # postprocess-room: склейка и mux
  master/
    backend/          # Master API, PostgreSQL, postprocess orchestration
    frontend/         # Web UI
prometheus/           # Alert rules
grafana/              # Provisioning, datasources, dashboards
videos/               # Записи на хосте
docker-compose.yaml   # Полный локальный стенд
```
