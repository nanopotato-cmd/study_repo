# CI/CD пайплайн для Burger App

```text
std-32-obvintsev-module5/
├── .gitlab-ci.yml          # CI/CD пайплайн
├── deploy.sh               # Скрипт деплоя на ВМ
├── package.json            # Зависимости Node.js
├── client/                 # Фронтенд (React + Vite)
│   └── src/
├── server/                 # Бэкенд (Express + Drizzle)
│   ├── index.ts
│   ├── db.ts
│   └── storage.ts
├── shared/                 # Общие типы и схема БД
│   └── schema.ts
├── dist/                   # Собранное приложение (генерируется)
└── README.md               # Документация
```

## 📄 .gitlab-ci.yml (построчный разбор)

### Переменные

```yaml
image: node:18-alpine                    # Docker-образ для всех stage

variables:
  MAJOR: 1                               # Мажорная версия
  MINOR: 0                               # Минорная версия
  PATCH: ${CI_PIPELINE_ID}               # Патч = ID пайплайна
  VERSION: "${MAJOR}.${MINOR}.${PATCH}"  # SemVer: 1.0.29413
  NEXUS_URL: https://nxs.praktikum-services.tech
  NEXUS_REPO: raw-hosted
```

### Stage: build

```yaml
build:
  stage: build
  script:
    - npm ci --cache .npm --prefer-offline   # Чистая установка зависимостей
    - npm run build                           # Сборка (Vite + esbuild)
  artifacts:                                  # Сохраняем результат
    paths:
      - dist/                                 # Собранный фронтенд и бэкенд
      - server/                               # Исходники бэкенда
      - shared/                               # Общие типы
      - client/                               # Исходники фронтенда
      - package.json                          # Для установки на ВМ
      - package-lock.json
      - drizzle.config.ts                     # Конфиг Drizzle
      - components.json
      - vite.config.ts
      - postcss.config.js
      - tailwind.config.ts
      - tsconfig.json
    expire_in: 1 week                         # Артефакт живёт 1 неделю
  cache:                                      # Кеш для ускорения
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - node_modules/
      - .npm/
```

### Stage: release

```yaml
release:
  stage: release
  image: alpine:3.20                         # Лёгкий образ
  dependencies:
    - build                                  # Забирает артефакты из build
  before_script:
    - apk add --no-cache curl tar            # Устанавливаем утилиты
  script:
    # Упаковываем всё в архив
    - tar -czvf burger-app-${VERSION}.tar.gz dist/ server/ shared/ client/ package.json package-lock.json drizzle.config.ts components.json vite.config.ts postcss.config.js tailwind.config.ts tsconfig.json
    # Загружаем в Nexus
    - curl -u "${NEXUS_USER}:${NEXUS_PASSWORD}" --upload-file burger-app-${VERSION}.tar.gz "${NEXUS_URL}/repository/${NEXUS_REPO}/${VERSION}/burger-app-${VERSION}.tar.gz"
```

### Stage: deploy

```yaml
deploy:
  stage: deploy
  image: alpine:3.20
  dependencies:
    - build
  before_script:
    - apk add --no-cache openssh-client postgresql-client curl tar
    - mkdir -p ~/.ssh
    - echo "${SSH_PRIVATE_KEY_B64}" | base64 -d > ~/.ssh/id_ed25519
    - chmod 600 ~/.ssh/id_ed25519
    - ssh -i ~/.ssh/id_ed25519 -o StrictHostKeyChecking=no -f -N -L 6432:rc1a-7f7o0828plfcfm7r.mdb.yandexcloud.net:6432 ${VM_USER}@${VM_IP}
  script:
    - echo "Running database migrations..."
    - psql "postgresql://...@localhost:6432/...?sslmode=require" -c "SELECT 1;"
    - curl -u "${NEXUS_USER}:${NEXUS_PASSWORD}" -o /tmp/burger-app-${VERSION}.tar.gz "${NEXUS_URL}/repository/${NEXUS_REPO}/${VERSION}/burger-app-${VERSION}.tar.gz"
    - scp ... /tmp/burger-app-${VERSION}.tar.gz ${VM_USER}@${VM_IP}:/home/${VM_USER}/
    - scp ... ./deploy.sh ${VM_USER}@${VM_IP}:/home/${VM_USER}/deploy.sh
    - ssh ... "chmod +x /home/${VM_USER}/deploy.sh && /home/${VM_USER}/deploy.sh ${VERSION}"
  rules:
    - when: manual                           # Ручной запуск
```


## 📄 deploy.sh (построчный разбор)

```bash
#!/bin/bash
set -xe                                    # Выход при ошибке, вывод команд

VERSION=$1                                 # Версия из аргумента
APP_DIR=/opt/burger-app                    # Директория установки

# Установка Node.js 20 (если нет)
if ! command -v node &> /dev/null; then
    curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
    sudo apt install -y nodejs
fi

# Создаём директорию
sudo mkdir -p ${APP_DIR}
sudo chown ${USER}:${USER} ${APP_DIR} -R

# Распаковываем артефакт
cd ${APP_DIR}
tar -xzvf /home/${USER}/burger-app-${VERSION}.tar.gz

# Создаём символическую ссылку client/src → src
ln -sf client/src src

# Устанавливаем зависимости (без dev)
npm ci

# Устанавливаем drizzle-kit для миграций
npm install -D drizzle-kit

# Создаём .env с переменными
cat > .env << EOF
DATABASE_URL=postgresql://user:pass%23%23f9%21fg@host:6432/db?sslmode=no-verify
PORT=3000
FEATURE_NEW_CHECKOUT=${FEATURE_NEW_CHECKOUT:-true}
FEATURE_DARK_MODE=${FEATURE_DARK_MODE:-true}
EOF

# Загружаем .env и выполняем миграции
export $(cat .env | xargs)
npx drizzle-kit push

# Запускаем приложение
nohup npm start > /var/log/burger-app.log 2>&1 &
```

## 🗄️ Миграции базы данных

### Как работают миграции в проекте

1. **Drizzle ORM** используется для работы с PostgreSQL
2. Схема описана в `shared/schema.ts`
3. Миграции выполняются через `drizzle-kit push`

### Команды для миграций

```bash
# Применить миграции (создать/обновить таблицы)
npx drizzle-kit push

# Сгенерировать файл миграции (если не используется push)
npx drizzle-kit generate

# Проверить состояние
npx drizzle-kit introspect
```

## 🔐 Работа с .env

Файл `.env` хранит переменные окружения для приложения. Он не должен храниться в Git (добавлен в `.gitignore`).

### Структура .env
```text
DATABASE_URL=postgresql://user:pass@host:6432/db?sslmode=no-verify
PORT=3000
FEATURE_NEW_CHECKOUT=true
FEATURE_DARK_MODE=true
```

## 📌 Ключевые моменты для запоминания

| Компонент       | Назначение                                            |
|-----------------|-------------------------------------------------------|
| GitLab CI       | Автоматизация сборки и деплоя                         |
| Nexus           | Хранилище артефактов (версии)                         |
| deploy.sh       | Скрипт развёртывания на ВМ                            |
| .env            | Переменные окружения (не в Git)                       |
| Drizzle         | ORM для миграций БД                                   |
| SSH-туннель     | Доступ к БД через ВМ                                  |
| SemVer          | Версионирование артефактов                            |
| Фича-флаги      | Управление функциями                                  |