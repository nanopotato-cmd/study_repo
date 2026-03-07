# Шпаргалки по Git

## Основные команды Git

### Базовые команды навигации
```bash
pwd                              # Посмотреть, где вы сейчас находитесь
ls -la                           # Посмотреть содержимое текущей папки
cd ~                             # Перейти в домашнюю папку
git status                       # Проверка статуса
```

### Настройка Git
```bash
git config --global user.name "Ваше Имя"
git config --global user.email "ваш@email.com"
```

### Работа с репозиторием
```bash
git init                                                 # Создать новый репозиторий
git clone https://github.com/user/repo.git               # Клонировать существующий
```

### Основные команды для коммитов
```bash
# Добавить файлы
git add filename.txt    # конкретный файл
git add .               # все файлы

# Сделать коммит
git commit -m "сообщение"

# Отправить на GitHub
git push origin master
```


| Команда | Описание | Пример |
|---------|----------|--------|
| `git init` | Создать репозиторий | `git init` |
| `git add` | Добавить файлы | `git add index.html` |
| `git commit` | Сохранить изменения | `git commit -m "фикс"` |


## Пример работы с ветками

```bash
git branch feature/new-feature # Создать новую ветку
git checkout feature/new-feature # Переключиться на ветку
git checkout -b feature/new-feature # Создать и сразу переключиться
```

## Полезные сокращения

Вот список моих любимых команд:
- `git status` - проверяю постоянно 👀
- `git log --oneline` - краткая история коммитов 📜
- `git diff` - смотрю изменения перед коммитом 🔍