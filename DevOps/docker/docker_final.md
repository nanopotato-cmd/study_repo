# Кратое описание финальной практики по Docker Compose

Это микросервисное приложение для онлайн-заказа пиццы. Оно состоит из:

**Frontend** — веб-интерфейс (React)
**Backend** — два сервиса: menu-service (меню) и order-service (заказы)
**Redis** — база данных для хранения состояния
**Nginx** — балансировщик и точка входа

## compose.yaml — оркестрация контейнеров

```yaml
# --- Имя проекта ---
name: galactic-pizza

# --- Определение сервисов ---
services:
  # ========== REDIS ==========
  # Хранилище данных для меню и заказов
  redis:
    image: redis:7-alpine
    command: >
      sh -c "redis-server /usr/local/etc/redis/redis.conf --requirepass $$(cat /run/secrets/redis_password)"
    volumes:
      - ./redis/redis.conf:/usr/local/etc/redis/redis.conf   # конфиг Redis
      - redis_data:/data                                     # постоянные данные
    secrets:
      - redis_password                                       # пароль из файла
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "$(cat /run/secrets/redis_password)", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - redis_network   # только Redis в своей сети

  # ========== MENU SERVICE ==========
  # Сервис меню (Node.js) — отдаёт список пицц
  menu-service:
    build: ./menu-service
    deploy:
      replicas: 2       # два экземпляра для балансировки
    environment:
      REDIS_HOST: redis
      REDIS_PORT: 6379
      PORT: 8080
      ORDER_SERVICE_URL: http://order-service:8080
    secrets:
      - redis_password
    depends_on:
      redis:
        condition: service_healthy   # ждём, пока Redis запустится
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend          # для nginx
      - redis_network    # для Redis

  # ========== ORDER SERVICE ==========
  # Сервис заказов (Spring Boot) — создание и отслеживание заказов
  order-service:
    build: ./order-service
    deploy:
      replicas: 2       # два экземпляра для балансировки
    environment:
      REDIS_HOST: redis
      REDIS_PORT: 6379
      SERVER_PORT: 8080
      MENU_SERVICE_URL: http://menu-service:8080
    secrets:
      - redis_password
    depends_on:
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8001/actuator/health"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend          # для nginx
      - redis_network    # для Redis

  # ========== FRONTEND ==========
  # React-приложение (статический сайт)
  frontend:
    build: ./frontend
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - frontend         # только фронтенд-сеть

  # ========== NGINX ==========
  # Балансировщик и точка входа
  nginx:
    build: ./nginx
    volumes:
      - ${PWD}/nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    ports:
      - "8888:80"        # порт на хосте:порт в контейнере
    depends_on:
      menu-service:
        condition: service_healthy
      order-service:
        condition: service_healthy
      frontend:
        condition: service_healthy
    networks:
      - frontend         # для общения с фронтом
      - backend          # для общения с API

# --- СЕТИ (изоляция) ---
networks:
  frontend:
    internal: true       # только nginx и frontend
  backend:
    internal: true       # nginx + menu + order
  redis_network:
    internal: true       # только Redis + menu + order

# --- ТОМА (постоянные данные) ---
volumes:
  redis_data:

# --- СЕКРЕТЫ (пароли) ---
secrets:
  redis_password:
    file: ./secrets/redis_password.txt
```

### 🔧 Сети (Network Isolation)

Каждая сеть изолирует свою группу сервисов. Это безопасно и соответствует принципу "наименьших привилегий".

| Сеть           | Сервисы                             | Назначение                              |
|----------------|-------------------------------------|-----------------------------------------|
| `frontend`     | nginx, frontend                     | Общение фронта с пользователем          |
| `backend`      | nginx, menu-service, order-service  | Бизнес-логика и API                     |
| `redis_network`| redis, menu-service, order-service  | Хранение данных                         |

### 🗄️ Redis

- Запускает Redis с паролем из секрета
- Монтирует конфиг и том для данных
- Проверяет здоровье через `redis-cli ping`
- Находится в сети `redis_network`

### 🍕 Menu Service

- Отдаёт список пицц (эндпоинт `/api/menu`)
- Читает данные из Redis
- Общается с `order-service` для создания заказов
- Масштабируется до 2 реплик
- Находится в двух сетях: `backend` (для nginx) и `redis_network` (для Redis)

### 📋 Order Service

- Обрабатывает заказы (эндпоинт `/api/orders`)
- Читает/пишет в Redis
- Общается с `menu-service` для проверки наличия
- Находится в тех же сетях, что и `menu-service`

### 🖥️ Frontend

- Отдаёт React-приложение (статику)
- Находится только в сети `frontend`
- Имеет healthcheck для проверки доступности

### 🌐 Nginx (балансировщик)

- Принимает все входящие запросы на порт 8888
- Направляет их по маршрутам:

1. / → frontend
2. /api/menu → menu-service
3. /api/orders → order-service

- Находится в двух сетях: frontend (для пользователей) и backend (для API)

## nginx.conf — маршрутизация и балансировка

```nginx
# ================================================================
#                      NGINX КОНФИГУРАЦИЯ
# ================================================================

worker_processes auto;
pid /var/run/nginx/nginx.pid;

events {}

http {
    # --- MIME типы (для статики) ---
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # --- Формат логов ---
    log_format galactic '[$time_local] 🌌 $remote_addr -> $upstream_addr '
                       '"$request" $status $body_bytes_sent '
                       'response_time=$request_time upstream_time=$upstream_response_time';

    access_log /var/log/nginx/access.log galactic;
    error_log /var/log/nginx/error.log warn;

    # --- DNS-резолвер Docker (для динамического обновления IP) ---
    resolver 127.0.0.11 valid=10s;

    # ============================================================
    #               UPSTREAM (бэкенды для балансировки)
    # ============================================================

    upstream menu_service_backend {
        server menu-service:8080;   # меню-сервис
    }

    upstream order_service_backend {
        server order-service:8080;  # заказ-сервис
    }

    upstream frontend_service {
        server frontend:80;         # фронтенд
    }

    # ============================================================
    #                    HTTP-СЕРВЕР
    # ============================================================

    server {
        listen 80;
        server_name localhost galactic-pizza;

        # ---------- API: МЕНЮ ----------
        # Прокси на menu-service
        location /api/menu {
            proxy_pass http://menu_service_backend;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_cache_bypass $http_upgrade;

            # Таймауты
            proxy_connect_timeout 30s;
            proxy_send_timeout 60s;
            proxy_read_timeout 60s;

            # Обработка ошибок бэкенда
            proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
            proxy_next_upstream_tries 3;
            proxy_next_upstream_timeout 30s;

            # Заголовки для идентификации
            proxy_set_header X-Service-Route "menu-service";
            add_header X-Upstream-Server $upstream_addr always;
        }

        # ---------- API: ЗАКАЗЫ ----------
        # Прокси на order-service
        location /api/orders {
            rewrite ^/api/orders/$ /api/orders break;
            proxy_pass http://order_service_backend;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_cache_bypass $http_upgrade;

            # Более долгие таймауты для заказов
            proxy_connect_timeout 30s;
            proxy_send_timeout 120s;
            proxy_read_timeout 120s;

            proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
            proxy_next_upstream_tries 2;
            proxy_next_upstream_timeout 60s;

            proxy_set_header X-Service-Route "order-service";
            add_header X-Upstream-Server $upstream_addr always;
        }

        # ---------- ФРОНТЕНД ----------
        # Отдача статики React
        location / {
            proxy_pass http://frontend:80;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_cache_bypass $http_upgrade;
        }

        # ---------- СТРАНИЦЫ ОШИБОК ----------
        error_page 404 /errors/404.html;
        error_page 500 502 503 504 /errors/50x.html;

        location = /errors/404.html {
            root /usr/share/nginx/html;
            internal;
        }

        location = /errors/50x.html {
            root /usr/share/nginx/html;
            internal;
        }
    }
}
```

### 🔄 Upstream (бэкенды)

Группирует серверы для балансировки. При масштабировании сюда можно добавить новые реплики.

### 🧭 Маршруты (location)

`/api/menu` - Проксирует запросы на menu-service
`/api/orders` - Проксирует запросы на order-service
`/` (фронтенд) - Отдаёт статику React-приложения

## ИТОГ

| Компонент       | Что сделано                                                        |
|-----------------|--------------------------------------------------------------------|
| Redis           | Настроен с паролем, healthcheck, томом для данных                  |
| menu-service    | Две реплики, healthcheck, связь с Redis и order-service            |
| order-service   | Две реплики, healthcheck, связь с Redis и menu-service             |
| frontend        | Статика, healthcheck                                               |
| nginx           | Балансировка, прокси, маршрутизация                                |
| Сети            | Три изолированные сети для безопасности                            |
| Секреты         | Пароль Redis вынесен в отдельный файл                              |

# Ключевые термины

| Термин            | Что значит                                                              |
|-------------------|-------------------------------------------------------------------------|
| Healthcheck       | Проверка, что сервис жив и готов принимать запросы                      |
| Replicas          | Количество копий сервиса для балансировки и отказоустойчивости          |
| Internal network  | Сеть, доступная только внутри Docker, не наружу                         |
| Secrets           | Конфиденциальные данные (пароли), не хранятся в коде                    |
| Proxy_pass        | Перенаправление запроса на другой сервер                                |