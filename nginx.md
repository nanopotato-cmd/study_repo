# Nginx — Конспект для DevOps

## Что такое Nginx?

**Nginx** (читается "Энжин-икс") — это:
- Веб-сервер (отдаёт HTML, CSS, JS, картинки)
- Обратный прокси-сервер (Reverse Proxy)
- Балансировщик нагрузки (Load Balancer)
- Кэширующий сервер

**Ключевая особенность:** Асинхронная архитектура с одним мастер-процессом и несколькими воркер-процессами. Это позволяет Nginx обрабатывать десятки тысяч соединений одновременно при низком потреблении памяти.

## Зачем нужен Nginx в веб-разработке?

| Задача | Решение от Nginx |
| :--- | :--- |
| **Отдача статики** (HTML, CSS, JS) | В 10-100 раз быстрее, чем через Python/Node.js |
| **Обратный прокси** | Скрывает внутренние сервисы (Flask, Django, Node.js) за одним портом 80/443 |
| **Балансировка нагрузки** | Распределяет запросы между несколькими серверами |
| **SSL/TLS терминация** | Обрабатывает HTTPS, а внутренние сервисы работают по HTTP |
| **Сжатие (gzip)** | Уменьшает размер ответов в 3-5 раз |
| **Кэширование** | Сохраняет часто запрашиваемые ответы в памяти/на диске |
| **Лимитирование запросов** | Защита от DDoS и брутфорса |
| **Виртуальные хосты** | Один Nginx обслуживает несколько сайтов на одном IP |

- **Master Process** — читает конфигурацию, запускает/останавливает воркеры
- **Worker Process** — обрабатывает клиентские соединения (каждый воркер — отдельный процесс)

В проекте sport_day Nginx играет роль обратного прокси-сервера (reverse proxy). Вот простая аналогия:

Представьте, что ваш сервер — это многоквартирный дом:

- **Nginx** = консьерж на входе
- **Flask API** = квартира 5000 (ваше Python-приложение)
- **HTML/CSS/JS** = квартира 80 (статический сайт)

Консьерж (Nginx):
- Смотрит, куда идёт посетитель
- Если просят `/` — отправляет в квартиру 80 (статический сайт)
- Если просят `/api/...` — отправляет в квартиру 5000 (Flask)
- Может запомнить часто посещаемые страницы (кэш)

```nginx
# server — это один виртуальный хост (один сайт на сервере)
server {
    # Какой порт слушать (80 — стандартный HTTP порт)
    listen 80;
    
    # На каком доменном имени/IP отвечать
    # localhost — для локального доступа
    # 192.168.0.135 — ваш IP в локальной сети
    server_name localhost 192.168.0.135;
    
    # Корневая директория для статических файлов (HTML, CSS, JS, картинки)
    root /var/www/sport_day/html;
    
    # Какой файл отдавать, если запросили директорию
    # Например, при запросе / отдаст /var/www/sport_day/html/index.html
    index index.html;
    
    # Блок location — обрабатывает определённый URL-путь
    # / — обрабатывает все запросы, которые НЕ начинаются с /api/
    location / {
        # Пытается найти запрошенный файл
        # Если файла нет — пробует директорию с тем же именем
        # Если ничего не найдено — возвращает 404
        try_files $uri $uri/ =404;
    }
    
    # /api/ — обрабатывает все запросы, начинающиеся с /api/
    # Это наш бекенд на Flask
    location /api/ {
        # Проксирует запрос на внутренний адрес
        # Flask запущен на localhost:5000
        proxy_pass http://localhost:5000;
        
        # Передаём оригинальный Host (имя сервера)
        # Flask узнает, по какому адресу к нему обратились
        proxy_set_header Host $host;
        
        # Передаём реальный IP клиента (а не IP Nginx)
        # Flask увидит IP пользователя, а не 127.0.0.1
        proxy_set_header X-Real-IP $remote_addr;
        
        # Передаём полную цепочку прокси (если их несколько)
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        
        # Передаём протокол (http/https)
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Увеличиваем таймауты для долгих запросов
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
    
    # Блок для статических файлов (кэширование на 1 день)
    location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
        expires 1d;
        add_header Cache-Control "public, immutable";
    }
    
    # Сжатие текстовых ответов (HTML, CSS, JS, JSON)
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml;
    gzip_min_length 1000;  # не сжимать файлы меньше 1KB
}
```

## Таблица директив Nginx

| Директива | Значение | Пример в вашем проекте |
| :--- | :--- | :--- |
| `listen` | Какой порт и протокол слушает сервер | `80` — стандартный HTTP порт |
| `server_name` | На какие доменные имена или IP-адреса отвечает этот блок | `localhost`, `192.168.0.135` |
| `root` | Корневая директория, где лежат статические файлы сайта | `/var/www/sport_day/html` |
| `index` | Какой файл отдавать по умолчанию при запросе директории | `index.html` |
| `try_files` | Порядок поиска файлов — проверяет существование по очереди | `$uri $uri/ =404` (сначала файл, потом папка, потом 404) |
| `location /` | Блок конфигурации для всех запросов, не подпавших под другие `location` | обрабатывает HTML, CSS, JS, картинки |
| `location /api/` | Блок конфигурации для URL, начинающихся с `/api/` | перенаправляет все API-запросы на Flask |
| `proxy_pass` | URL, на который нужно перенаправить (проксировать) входящий запрос | `http://localhost:5000` (Flask на порту 5000) |
| `proxy_set_header` | Добавляет или изменяет HTTP-заголовки при проксировании | передаёт реальный IP клиента, оригинальный Host и протокол |
| `expires` | Устанавливает время кэширования ответа в браузере клиента | `1d` — кэшировать статику (CSS, JS, картинки) на 1 день |
| `gzip` | Включает сжатие ответов перед отправкой клиенту | `on` — уменьшает размер трафика в 3-5 раз |
| `gzip_types` | Указывает, какие MIME-типы ответов нужно сжимать | `text/plain text/css application/json application/javascript` |
| `gzip_min_length` | Минимальный размер ответа в байтах, при котором применяется сжатие | `1000` — не сжимать файлы меньше 1KB |
| `access_log` | Указывает файл для записи логов всех входящих запросов | `/var/log/nginx/sport_day_access.log` |
| `error_log` | Указывает файл для записи логов ошибок | `/var/log/nginx/sport_day_error.log` |
| `proxy_connect_timeout` | Таймаут на установление соединения с проксируемым сервером | `60s` — ждать подключения к Flask не более 60 секунд |
| `proxy_send_timeout` | Таймаут на отправку данных проксируемому серверу | `60s` |
| `proxy_read_timeout` | Таймаут на получение ответа от проксируемого сервера | `60s` |
| `proxy_buffering` | Включает/отключает буферизацию ответа от проксируемого сервера | `off` — данные идут клиенту сразу, без накопления |
| `add_header` | Добавляет произвольный заголовк в HTTP-ответ | `Cache-Control "public, immutable"` |
| `server_tokens` | Показывать ли версию Nginx в заголовке `Server` | `off` — скрывает версию для безопасности |
| `include` | Подключает внешний файл конфигурации | `include /etc/nginx/mime.types` |
| `return` | Возвращает клиенту HTTP-статус и (опционально) текст | `return 404 "Not found";` |
| `rewrite` | Изменяет URL по регулярному выражению (перенаправление) | `rewrite ^/old/(.*) /new/$1 permanent;` |


##  Полезные команды для работы с Nginx

```bash
# Проверить конфигурацию на ошибки
sudo nginx -t

# Перезагрузить конфигурацию без остановки сервера
sudo systemctl reload nginx

# Полностью перезапустить Nginx
sudo systemctl restart nginx

# Посмотреть статус
sudo systemctl status nginx

# Посмотреть логи доступа
sudo tail -f /var/log/nginx/access.log

# Посмотреть логи ошибок
sudo tail -f /var/log/nginx/error.log
```

Итог: Nginx — это "входная дверь" вашего сервера. Он принимает все запросы и решает, кому их передать: статическому сайту или Python-API. Это стандартная практика в продакшене. 🚀


## Типовые схемы работы Nginx

1. **Простой веб-сервер (только статика)**
```
Пользователь → Nginx → HTML/CSS/JS/картинки
```

2. **Обратный прокси (ваш случай)**
```
Пользователь → Nginx (порт 80) → Flask (порт 5000)
                        ↘ HTML/CSS (статический сайт)
```

3. **Балансировка нагрузки**
```
                    ┌─→ Flask 1 (порт 5001)
Пользователь → Nginx ┼─→ Flask 2 (порт 5002)
                    └─→ Flask 3 (порт 5003)
```

4. **SSL-терминация (HTTPS)**
```
Пользователь → Nginx (порт 443, HTTPS) → Flask (порт 5000, HTTP)
```

## Где лежат конфиги Nginx

| Путь                                   | Назначение                               |
|----------------------------------------|------------------------------------------|
| `/etc/nginx/nginx.conf`                | Главный конфиг                           |
| `/etc/nginx/sites-available/`          | Все доступные конфиги сайтов             |
| `/etc/nginx/sites-enabled/`            | Симлинки на активные сайты               |
| `/etc/nginx/conf.d/`                   | Дополнительные конфиги                   |
| `/var/log/nginx/access.log`            | Лог всех запросов                        |
| `/var/log/nginx/error.log`             | Лог ошибок                               |

# Клиент → Nginx → Flask → БД: Полная цепочка запроса

## От нажатия кнопки до данных на экране

Представьте, что вы зашли на сайт примерной доски объявлений. Всё, что происходит внутри системы, можно разбить на понятные этапы.

```mermaid
sequenceDiagram
    participant Браузер as 🖥️ Браузер (клиент)
    participant DNS as 🌐 DNS-сервер
    participant Nginx as ⚙️ Nginx (веб-сервер)
    participant Flask as 🐍 Flask (приложение)
    participant БД as 🗄️ PostgreSQL (база данных)

    Note over Браузер,БД: 1. Пользователь вводит адрес
    
    Браузер->>DNS: "Где находится ads.example.com?"
    DNS-->>Браузер: "По адресу 185.230.120.15"
    
    Браузер->>Nginx: "GET /ads/123"
    Note right of Nginx: 2. Nginx обрабатывает запрос
    
    alt Статика (CSS/JS/картинки)
        Nginx-->>Браузер: Отдаёт файл напрямую
    else Динамика (API)
        Nginx->>Flask: Передаёт запрос
        Note right of Flask: 3. Flask обрабатывает логику
        
        Flask->>БД: "SELECT * FROM ads WHERE id=123"
        БД-->>Flask: Возвращает данные
        
        Flask-->>Nginx: JSON с объявлением
        Nginx-->>Браузер: JSON-ответ
        
        Note right of Браузер: 4. Браузер отображает страницу
        Браузер->>Браузер: Строит HTML из JSON
    end
```

## Шаг 1: Клиент (браузер)

```javascript
// Вы нажали на объявление №123
// Браузер выполняет примерно такой код:

// 1. Формирует запрос
fetch('https://ads.example.com/ads/123', {
    method: 'GET',
    headers: {
        'Accept': 'application/json',
        'Accept-Language': 'ru-RU'
    }
})

// 2. Ждёт ответ от сервера
.then(response => {
    // 3. Получает JSON с объявлением
    return response.json();
})
.then(ad => {
    // 4. Создаёт HTML из полученных данных
    document.getElementById('title').innerText = ad.title;
    document.getElementById('price').innerText = ad.price + ' ₽';
    document.getElementById('description').innerText = ad.description;
});
```
## Шаг 2: DNS — интернет-телефонная книга

Простыми словами: DNS похож на телефонную книгу. Вы помните имя "ads.example.com", а DNS говорит "Звоните по номеру 185.230.120.15".

## Шаг 3: Nginx — главный привратник

### Что делает Nginx?

**Nginx** — это первое, что встречает запрос от браузера. Он решает:

Вопрос Nginx                       | Что решает
-----------------------------------|---------------------------------------------------
"Это картинка или API?"            | Статику (CSS, JS, картинки) отдаю сразу
"Сколько таких запросов?"          | Защищаю от DDoS (ограничение частоты)
"Кто стучится?"                    | Проверяю, не заблокирован ли IP
"Какой размер файла?"              | Сжимаю (gzip) для экономии трафика

### Конфигурация Nginx (упрощённо)

```nginx
server {
    listen 80;
    server_name ads.example.com;
    
    # Статика — отдаём напрямую
    location /static/ {
        alias /var/www/static/;
        expires 1y;  # кэшируем на год
    }
    
    # Картинки объявлений
    location /uploads/ {
        alias /var/www/uploads/;
        expires 1M;  # кэшируем на месяц
    }
    
    # API — передаём Flask
    location /api/ {
        proxy_pass http://flask_app:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
    
    # Защита от ботов (rate limiting)
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
}
```

Простыми словами: Nginx — как швейцар в дорогом отеле. Он встречает гостей, проверяет, кто они, маленькие подарки (картинки) отдаёт сам, а важных гостей (API запросы) провожает к нужному человеку (Flask).

## Шаг 4: Flask — умный менеджер

### Что делает Flask?
**Flask** запускает Python-код, который:

- Проверяет, правильный ли запрос
- Берёт данные из базы
- Формирует ответ

Простыми словами: Flask — как менеджер ресторана. Он принимает заказ от швейцара (Nginx), проверяет, можно ли его выполнить (авторизация), идёт на кухню (БД) за ингредиентами, готовит блюдо (JSON) и отдаёт швейцару, чтобы тот передал клиенту.

## Шаг 5: БД (PostgreSQL) — архивный склад

Простыми словами: БД — как огромный архивный склад. Flask присылает накладную (SQL запрос), сотрудники склада (планировщик, индексы) ищут нужную полку, берут коробку и выдают содержимое.

# Пример с реальным кодом (всё вместе)
## Файл: nginx.conf
```nginx
server {
    listen 80;
    server_name ads.example.com;
    
    location /api/ {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```
## Файл: app.py (Flask)
```python
from flask import Flask, jsonify
from flask_sqlalchemy import SQLAlchemy

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'postgresql://user:pass@localhost/ads'
db = SQLAlchemy(app)

class Ad(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(200))
    price = db.Column(db.Integer)

@app.route('/api/ads/<int:ad_id>')
def get_ad(ad_id):
    ad = db.session.get(Ad, ad_id)
    return jsonify({'title': ad.title, 'price': ad.price})

if __name__ == '__main__':
    app.run(port=5000)
```

## Запрос из браузера (JavaScript)
```javascript
// Браузер отправляет запрос
fetch('/api/ads/123')
    .then(response => response.json())
    .then(ad => {
        document.getElementById('title').innerText = ad.title;
        document.getElementById('price').innerText = ad.price + ' ₽';
    });
```

## Что происходит под капотом (по шагам)
```bash
# 1. Браузер спрашивает DNS
nslookup ads.example.com
# → 185.230.120.15

# 2. Браузер отправляет HTTP-запрос
curl -v http://185.230.120.15/api/ads/123

# 3. Nginx принимает запрос (логи)
nginx log: 192.168.1.1 - - [05/Apr/2024:12:05:23] "GET /api/ads/123 HTTP/1.1" 200

# 4. Flask обрабатывает (логи)
[2024-04-05 12:05:23] GET /api/ads/123 -> returned 200 in 12ms

# 5. PostgreSQL выполняет запрос (логи)
LOG: execute <unnamed>: SELECT ads.id, ads.title, ads.price 
FROM ads WHERE ads.id = $1
DETAIL: parameters: $1 = '123'
LOG: duration: 3.234 ms

# 6. Браузер получает JSON и отображает
{"title":"Красивый диван","price":15000}
```

## Что DevOps должен контролировать

### 1. Nginx
```bash
# Проверка соединения с Flask
curl -I http://localhost:5000/health

# Проверка конфигурации
nginx -t

# Просмотр логов
tail -f /var/log/nginx/access.log
tail -f /var/log/nginx/error.log

# Следим за ошибками
grep "502" /var/log/nginx/error.log
```

### 2. Flask

```python
# Добавить проверку здоровья для Nginx
@app.route('/health')
def health():
    # Проверяем БД
    db.session.execute('SELECT 1')
    return {'status': 'ok'}, 200

# Логи в JSON (для ELK)
import logging
import json

class JSONFormatter(logging.Formatter):
    def format(self, record):
        return json.dumps({
            'time': self.formatTime(record),
            'level': record.levelname,
            'message': record.getMessage(),
            'path': record.pathname,
            'line': record.lineno
        })
```

### 3. База данных

```sql
-- Самые медленные запросы
SELECT query, mean_time, calls 
FROM pg_stat_statements 
ORDER BY mean_time DESC 
LIMIT 10;

-- Заблокированные запросы
SELECT pid, usename, query, state 
FROM pg_stat_activity 
WHERE state = 'blocked';

-- Размер таблицы
SELECT pg_size_pretty(pg_total_relation_size('ads'));
```

### 4. Полное логирование запроса

## Резюме одной картинкой

```mermaid
flowchart LR
    subgraph Frontend
        USER[👤 Пользователь<br/>нажал кнопку]
        BROWSER[🌐 Браузер<br/>формирует запрос]
    end
    
    subgraph Network
        DNS[📖 DNS<br/>185.230.120.15]
    end
    
    subgraph Backend
        NGINX[⚙️ Nginx<br/>привратник]
        
        subgraph App
            FLASK1[🐍 Flask<br/>экземпляр #1]
            FLASK2[🐍 Flask<br/>экземпляр #2]
        end
        
        DB[🗄️ PostgreSQL<br/>данные]
    end
    
    USER --> BROWSER
    BROWSER --> DNS
    DNS --> NGINX
    NGINX --> FLASK1
    NGINX --> FLASK2
    FLASK1 --> DB
    FLASK2 --> DB
    
    DB -.->|JSON| NGINX
    NGINX -.->|ответ| BROWSER
    BROWSER -.->|отображение| USER
```

**Самое важное запомнить:**

1. **Nginx** быстро отдаёт статику и защищает от атак
2. **Flask** выполняет бизнес-логику и общается с БД
3. **БД** хранит данные и должна быть быстрой
4. Вся эта цепочка занимает ~50 мс (незаметно)
5. DevOps должен следить за каждым звеном этой цепи