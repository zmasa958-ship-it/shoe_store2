# Установка и запуск ShoeStore

## Требования

- Windows 10/11
- Docker Desktop с Linux containers и Docker Compose
- PostgreSQL 18 (запускается в Docker, отдельная установка не требуется)

## Состав комплекта поставки

```text
ShoeStore-release/
├── INSTALL.md
├── application/
│   └── ShoeStore/
│       ├── ShoeStore.exe
│       ├── config.ini.example
│       └── config.ini            (создается при установке)
├── database/
│   └── shoe_store.sql
└── deployment/
    ├── docker-compose.yml
    ├── .env.example
    └── .env                      (создается при установке)
```

## Подготовка базы

Все команды `docker compose` выполняются в PowerShell из папки `deployment` комплекта поставки.

1. Создайте файл `deployment/.env` по образцу `deployment/.env.example`:

   ```powershell
   Copy-Item .env.example .env
   ```

2. Откройте `.env` и укажите локальный пароль. Не публикуйте этот файл и не добавляйте его в Git.
3. Запустите контейнеры:

   ```powershell
   docker compose up -d
   ```

4. Дождитесь, пока база станет доступна (статус `Up` или `Up (healthy)`):

   ```powershell
   docker compose ps
   docker compose exec db pg_isready -U postgres -d shoe_store
   ```

   Ожидаемый ответ: `accepting connections`.

5. При новой пустой базе импортируйте SQL-файл (подробнее в разделе «Импорт базы данных»).

## Импорт базы данных

Выполните из папки `deployment`:

```powershell
docker compose cp ../database/shoe_store.sql db:/tmp/shoe_store.sql
docker compose exec db psql -U postgres -d shoe_store -v ON_ERROR_STOP=1 -f /tmp/shoe_store.sql
```

Проверьте, что данные загружены:

```powershell
docker compose exec db psql -U postgres -d shoe_store -c "\dt"
```

### Особенности SQL-импорта

- **Импорт выполняется только в пустую базу.** Повторный импорт в базу, где уже есть таблицы, приведет к ошибкам вида `relation ... already exists` и дублированию данных.
- **Файл копируется внутрь контейнера** командой `docker compose cp`, а не передается через `<` или `>`. Так в PowerShell не возникает проблем с кодировкой кириллицы.
- **Файл `shoe_store.sql` должен быть в кодировке UTF-8.** Иначе названия товаров, поставщиков и пользователей отобразятся неправильно.
- **Флаг `-v ON_ERROR_STOP=1`** останавливает импорт на первой ошибке. Без него `psql` продолжит выполнение и оставит базу в частично заполненном состоянии.
- **Если импорт прошел с ошибкой,** очистите схему и повторите импорт. Эта команда удаляет все данные в базе, поэтому выполняйте ее только если нужных данных там нет:

  ```powershell
  docker compose exec db psql -U postgres -d shoe_store -c "DROP SCHEMA public CASCADE; CREATE SCHEMA public;"
  ```

## Настройка приложения

1. В папке `application/ShoeStore` скопируйте `config.ini.example` в `config.ini`:

   ```powershell
   Copy-Item config.ini.example config.ini
   ```

2. Укажите в `config.ini` параметры подключения:
   - адрес сервера: `127.0.0.1`
   - порт: `5433` (опубликованный порт, проброшенный на порт `5432` контейнера `db`)
   - имя базы данных: `shoe_store`
   - пользователь: `postgres`
   - пароль: тот же, что указан в `deployment/.env`

## Запуск

Откройте `application/ShoeStore/ShoeStore.exe`.

## Проверка

Убедитесь, что окно открылось и каталог товаров загрузился. Если появилось сообщение об ошибке подключения, воспользуйтесь разделом «Диагностика проблем».

## Диагностика проблем

### Приложение не подключается к базе данных

- Проверьте, что Docker Desktop запущен.
- Проверьте статус контейнеров: `docker compose ps` (должен быть `Up (healthy)`).
- Проверьте, что порт `5433` не занят другим приложением.
- Проверьте параметры в `config.ini` (хост, порт, пароль).

### Ошибка "relation 'tovar' does not exist"

- База данных пуста. Выполните импорт SQL-скрипта (см. раздел "Импорт базы данных").

### Ошибка "No module named flet" или подобная

- Приложение запущено не из EXE, а из исходного кода. Убедитесь, что вы запускаете `ShoeStore.exe`, а не `main.py`.

### Приложение не запускается из другой папки

- Убедитесь, что вы скопировали **всю папку** `ShoeStore`, а не только EXE-файл.

## Резервное копирование

Перед любыми действиями, которые могут затронуть данные, создайте резервную копию (из папки `deployment`):

```powershell
New-Item -ItemType Directory -Force ..\backups
docker compose exec db pg_dump -U postgres -d shoe_store --clean --if-exists -f /tmp/backup.sql
docker compose cp db:/tmp/backup.sql ../backups/shoe_store_backup.sql
```

Восстановление из копии:

```powershell
docker compose cp ../backups/shoe_store_backup.sql db:/tmp/restore.sql
docker compose exec db psql -U postgres -d shoe_store -v ON_ERROR_STOP=1 -f /tmp/restore.sql
```

## Остановка

Для остановки сервисов без потери данных (из папки `deployment`):

```powershell
docker compose stop
```

> **Важно:** не используйте `docker compose down -v` без резервной копии. Флаг `-v` удаляет том `db_data` и все данные в базе.ааа
