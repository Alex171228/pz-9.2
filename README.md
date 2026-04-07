# Практическое задание 9
## Шишков А.Д. ЭФМО-02-25
## Тема
Реализация распределённого кэша (Redis cluster)

## Цель
Освоить внедрение распределённого кэша в backend-приложение на Go и реализовать стратегию cache-aside с использованием Redis, корректного TTL, jitter и устойчивого поведения сервиса при недоступности кэша.

---

## 1. Зачем backend-приложению кэширование

Серверное приложение получает данные по запросам на чтение, обращаясь к базе данных. Если запросов много, а одни и те же записи читаются повторно, БД становится узким местом. Кэш — промежуточное хранилище уже полученных результатов, которое:

- **ускоряет ответ** — чтение из Redis на порядок быстрее, чем из PostgreSQL;
- **снижает нагрузку на БД** — повторные запросы не доходят до базы;
- **не заменяет БД** — кэш временный, данные могут истечь или быть удалены.

Redis подходит для этой роли как быстрый in-memory key-value store с поддержкой TTL и атомарных операций.

---

## 2. Что такое cache-aside

Cache-aside — стратегия кэширования, при которой приложение само управляет кэшем:

```
GET /v1/tasks/{id}
        │
        ▼
┌─── Redis GET ───┐
│                 │
│  hit?           │
│  ├─ да → ответ  │
│  └─ нет ────────┼──► БД → ответ
│                 │       │
│                 │   Redis SET (TTL)
└─────────────────┘
```

1. Приложение проверяет ключ в Redis
2. **Cache hit** — данные найдены, отдаём из кэша
3. **Cache miss** — идём в БД, результат кладём в Redis с TTL
4. При обновлении/удалении — инвалидируем ключ

Redis не является источником истины — это ускоритель чтения. При его недоступности сервис продолжает работать через БД.

---

## 3. Структура кэш-слоя

```
services/tasks/internal/cache/
  redis.go   — создание клиента и ping
  keys.go    — формирование ключей
  ttl.go     — TTL с jitter
```

### Формирование ключей

```go
func TaskByIDKey(id string) string {
    return fmt.Sprintf("tasks:task:%s", id)
}
```

Ключи предсказуемы, уникальны и легко инвалидируются.

### TTL с jitter

```go
func TTLWithJitter(base, jitter time.Duration) time.Duration {
    if jitter <= 0 {
        return base
    }
    extra := time.Duration(rand.Int63n(int64(jitter) + 1))
    return base + extra
}
```

- **TTL** = 120 секунд — время жизни записи в кэше
- **Jitter** = 0–30 секунд — случайная добавка, предотвращающая одновременное истечение множества ключей (cache stampede)

Итоговый TTL каждой записи: от 120 до 150 секунд.

---

## 4. Реализация cache-aside в сервисном слое

Файл `services/tasks/internal/service/task.go`:

```go
func (s *TaskService) GetByID(ctx context.Context, id string) (*Task, error) {
    key := cache.TaskByIDKey(id)

    if s.redis != nil {
        cached, err := s.redis.Get(ctx, key).Result()
        if err == nil {
            var t Task
            if err := json.Unmarshal([]byte(cached), &t); err == nil {
                s.log.Info("cache hit", zap.String("key", key))
                return &t, nil
            }
        } else if errors.Is(err, redis.Nil) {
            s.log.Info("cache miss", zap.String("key", key))
        } else {
            s.log.Warn("redis read error", zap.String("key", key), zap.Error(err))
        }
    }

    task, err := s.repo.GetByID(id)
    if err != nil {
        return nil, err
    }

    if s.redis != nil {
        bytes, _ := json.Marshal(task)
        ttl := cache.TTLWithJitter(s.ttl, s.ttlJitter)
        s.redis.Set(ctx, key, bytes, ttl)
        s.log.Info("cache set", zap.String("key", key), zap.Duration("ttl", ttl))
    }

    return task, nil
}
```

### Инвалидация кэша

При изменении или удалении задачи кэш инвалидируется:

```go
func (s *TaskService) invalidateCache(ctx context.Context, id string) {
    if s.redis == nil {
        return
    }
    key := cache.TaskByIDKey(id)
    s.redis.Del(ctx, key)
    s.log.Info("cache invalidated", zap.String("key", key))
}
```

Вызывается в `Update()` и `Delete()` после успешного изменения в БД.

---

## 5. Подключение Redis

### Конфигурация через переменные окружения

| Переменная | Значение по умолчанию | Описание |
|---|---|---|
| `REDIS_ADDR` | `localhost:6379` | Адрес Redis |
| `CACHE_TTL` | `120s` | Базовый TTL кэша |
| `CACHE_TTL_JITTER` | `30s` | Максимальный jitter |

### Инициализация в main.go

```go
redisClient = cache.NewRedisClient(redisAddr, "", 2*time.Second, 2*time.Second, 2*time.Second)
if err := cache.Ping(context.Background(), redisClient); err != nil {
    log.Warn("redis unavailable at startup, caching disabled", zap.Error(err))
    redisClient = nil
} else {
    log.Info("connected to redis", zap.String("addr", redisAddr))
}
```

Таймауты подключения — 2 секунды. Если Redis недоступен при старте, сервис работает без кэша.

---

## 6. Docker Compose

Redis добавлен в `deploy/docker-compose.yml`:

```yaml
redis:
  image: redis:7.4-alpine
  container_name: tasks_redis
  ports:
    - "6379:6379"
  healthcheck:
    test: ["CMD", "redis-cli", "ping"]
    interval: 3s
    timeout: 3s
    retries: 10
```

Сервис tasks зависит от Redis:

```yaml
tasks:
  environment:
    REDIS_ADDR: "redis:6379"
    CACHE_TTL: "120s"
    CACHE_TTL_JITTER: "30s"
  depends_on:
    redis:
      condition: service_healthy
```

---

## 7. Деградация при недоступности Redis

Сервис **не падает** при недоступности Redis:

- При старте: если ping не прошёл — `redisClient = nil`, кэширование отключается
- При работе: ошибки Redis логируются, запрос обслуживается из БД
- Все проверки `if s.redis != nil` гарантируют, что без Redis сервис работает штатно

---

## 8. Запуск и проверка

### Запуск

```bash
cd deploy
docker compose up -d --build
```

### Сценарий 1: cache miss → cache set → cache hit

```bash
# Первый запрос — cache miss, данные из БД
curl http://localhost:8082/v1/tasks/{id}

# Второй запрос — cache hit, данные из Redis
curl http://localhost:8082/v1/tasks/{id}
```

<img width="1897" height="90" alt="image" src="https://github.com/user-attachments/assets/acf68f41-05d4-4904-b7b8-a3dbab1e9e3b" /> 

<img width="1864" height="33" alt="image" src="https://github.com/user-attachments/assets/45df523a-f5c4-47cd-b104-602e05ee68cc" /> 


### Сценарий 2: инвалидация после PATCH

```bash
curl -X PATCH http://localhost:8082/v1/tasks/{id} \
  -H "Content-Type: application/json" \
  -d '{"title":"Updated"}'

# Следующий GET — снова cache miss
curl http://localhost:8082/v1/tasks/{id}
```

<img width="1891" height="61" alt="image" src="https://github.com/user-attachments/assets/e1701484-5649-4a4c-8ba9-867c3e29ee6d" /> 

<img width="1889" height="93" alt="image" src="https://github.com/user-attachments/assets/c69a2f64-907b-476e-8cba-a51f3d8568e3" /> 

### Сценарий 3: инвалидация после DELETE

```bash
curl -X DELETE http://localhost:8082/v1/tasks/{id}

# GET вернёт 404
curl http://localhost:8082/v1/tasks/{id}
```
<img width="1901" height="112" alt="image" src="https://github.com/user-attachments/assets/764fe499-2a88-4b14-8f56-a7f160d652ac" /> 

<img width="1894" height="171" alt="image" src="https://github.com/user-attachments/assets/eebbd0aa-747b-4463-abf4-2a730618449b" /> 

### Сценарий 4: fallback при остановленном Redis

```bash
docker compose stop redis

# Сервис продолжает отвечать через БД
curl http://localhost:8082/v1/tasks/{id}
```

<img width="1903" height="379" alt="image" src="https://github.com/user-attachments/assets/3eb24e6b-6c89-4ba2-82c4-eb666dabf352" /> 

Для проверки можно использовать postman коллекцию https://github.com/Alex171228/pz-9.2/blob/main/pz9_postman_collection.json
---

## 9. Демонстрация

**Запуск Postman-коллекции:**

<img width="1398" height="638" alt="image" src="https://github.com/user-attachments/assets/ebac6637-b371-47fc-afce-3e2959b5822f" /> 

<img width="1398" height="640" alt="image" src="https://github.com/user-attachments/assets/4b11ed05-f214-4b24-aa4e-3528f3ae3713" /> 

<img width="1398" height="158" alt="image" src="https://github.com/user-attachments/assets/a1afee58-ece3-4573-8a50-f1de76c4b8aa" /> 
