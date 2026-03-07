# Шпаргалки по Git

## Основные команды Git

### Настройка

```bash
pwd    # Посмотреть, где вы сейчас находитесь
ls -la # Посмотреть содержимое текущей папки
cd ~   # Перейти в домашнюю папку (чаще всего репозиторий создается здесь)
git status # Проверка статуса
```

```bash
git config --global user.name "<ваше_имя>"
git config --global user.email "<адрес_почты@email.com>"
```

### Работа с репозиторием

```bash
git init # Создать новый репозиторий
git clone https://github.com/user/repo.git # Клонировать существующий
```

### Коммиты

```bash
# Добавить файлы
git add filename.txt    # конкретный файл
git add .               # все файлы

# Сделать коммит
git commit -m "сообщение"

# Отправить на GitHub
git push origin master


| Команда | Описание | Пример |
|---------|----------|--------|
| `git init` | Создать репозиторий | `git init` |
| `git add` | Добавить файлы | `git add index.html` |
| `git commit` | Сохранить изменения | `git commit -m "фикс"` |


## Пример работы с ветками

```bash
# Создать новую ветку
git branch feature/new-feature

# Переключиться на ветку
git checkout feature/new-feature

# Создать и сразу переключиться
git checkout -b feature/new-feature
```

## Полезные сокращения

Вот список моих любимых команд:
- `git status` - проверяю постоянно 👀
- `git log --oneline` - краткая история коммитов 📜
- `git diff` - смотрю изменения перед коммитом 🔍

```bash
git config --global user.name "<ваше_имя>"
```