# Hop & Barley

Інтернет-магазин товарів для домашнього пивоваріння: каталог із фільтрами, кошик у сесії, оформлення замовлення з доставкою Новою Поштою, особистий кабінет, відгуки після покупки, staff-панель і REST API з документацією.

**Стек:** Python 3.12, Django 6.1, Django REST Framework, SimpleJWT, drf-spectacular, PostgreSQL 17, uv, Gunicorn, WhiteNoise, Caddy, Docker Compose, GitHub Actions.

## Що всередині

| Розділ | Адреса |
|---|---|
| Каталог, фільтри, пошук, сортування | `/` |
| Товар і відгуки | `/product/<slug>/` |
| Кошик та оформлення замовлення | `/cart/`, `/checkout/` |
| Кабінет: історія замовлень, профіль, адреса Нової Пошти | `/accounts/` |
| Staff-панель: товари й аналітика | `/staff/products/`, `/staff/dashboard/` |
| Django admin | `/admin/` |
| REST API | `/api/` |
| Swagger / ReDoc | `/api/docs/`, `/api/redoc/` |
| Healthcheck | `/health/` |

## Запуск локально без Docker

### Вимоги

- Python 3.12
- [uv](https://docs.astral.sh/uv/)
- PostgreSQL 17 або сумісна версія

### 1. Підготуй налаштування

Windows PowerShell:

```powershell
Copy-Item .env.example .env.local
```

Відкрий `.env.local` і налаштуй підключення до локального PostgreSQL:

```dotenv
DATABASE_URL=postgresql://USER:PASSWORD@localhost:5432/final3
```

Заміни `USER` і `PASSWORD` на фактичні облікові дані PostgreSQL. Переконайся, що база даних `final3` уже створена.

> `.env.example` містить адресу `db`, яка використовується в Docker Compose. Для локального запуску без Docker потрібно використовувати `localhost`.

Не додавай `.env.local` до Git: файл може містити паролі й секретні ключі.

### 2. Встанови залежності

```powershell
uv sync
```

### 3. Застосуй міграції

```powershell
uv run python manage.py migrate
```

### 4. За потреби створи демонстраційні дані

```powershell
uv run python manage.py seed_shop
```

Команда призначена для заповнення магазину демонстраційними товарами, покупцями, замовленнями та відгуками.

### 5. Створи адміністратора

```powershell
uv run python manage.py createsuperuser
```

### 6. Запусти сервер

```powershell
uv run python manage.py runserver
```

Відкрий у браузері: http://127.0.0.1:8000/

Демо-покупець, якщо його створено командою `seed_shop`:

- Email: `olena@example.com`
- Пароль: `demo-pass-123`

Вхід доступний за email або логіном.

## Запуск локально через Docker

Потрібні Docker Desktop із запущеним Docker Engine та Docker Compose.

### 1. Створи файл налаштувань

```powershell
Copy-Item .env.example .env.dev
```

У `.env.dev` перевір параметри PostgreSQL. Для Docker Compose хост бази даних має бути `db`, наприклад:

```dotenv
DATABASE_URL=postgresql://postgres:postgres@db:5432/final3
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_DB=final3
```

Це демонстраційні значення. Для постійного або спільного середовища зміни пароль на надійний і узгодь його з `DATABASE_URL`.

### 2. Запусти контейнери

```powershell
docker compose up --build
```

Застосунок буде доступний за адресою http://localhost:8000/

Compose очікує готовності PostgreSQL, після чого контейнер `web` застосовує міграції та запускає Django development server.

Якщо порт `5432` уже зайнятий локальним PostgreSQL, Docker може не запуститися через конфлікт портів. У такому разі зупини локальну службу PostgreSQL або зміни публікацію порту в `docker-compose.yml`.

Зупинити контейнери:

```powershell
docker compose down
```

Щоб також видалити дані бази з Docker volume, виконай `docker compose down -v`. **Це видалить дані PostgreSQL у відповідному Docker volume.**

## Перевірки

Запускай команди з кореня проєкту в активному середовищі Python.

Перевірка стилю коду:

```powershell
uv run ruff check .
uv run ruff format --check .
```

Перевірка типів:

```powershell
uv run mypy .
```

Тести:

```powershell
uv run pytest
```

Перевірка конфігурації Django для production:

```powershell
uv run python manage.py check --deploy
```

Production-перевірка може повідомляти про відсутні або небезпечні налаштування, якщо команда запускається з локальним профілем. Її результат потрібно оцінювати з урахуванням фактичного середовища.

CI запускає перевірки, налаштовані у workflow GitHub Actions.

## Production

### Архітектура

`Браузер → Caddy (HTTPS, /media/) → Gunicorn (Django + WhiteNoise) → PostgreSQL`

Caddy приймає зовнішні HTTP/HTTPS-запити, обслуговує медіафайли та передає інші запити застосунку через Gunicorn.

### 1. Підготуй сервер

Потрібні:

- VPS із встановленим Docker Engine та Docker Compose;
- домен, DNS-запис `A` якого вказує на IP сервера;
- доступність портів `80` і `443` з інтернету.

### 2. Отримай проєкт

На сервері виконай:

```bash
git clone YOUR_REPOSITORY_URL shop
cd shop
cp .env.example .env.production
```

Заміни `YOUR_REPOSITORY_URL` на справжню адресу Git-репозиторію.

### 3. Налаштуй `.env.production`

Укажи реальні значення для таких змінних:

| Змінна | Що вказати |
|---|---|
| `DJANGO_SETTINGS_MODULE` | `config.settings.prod` |
| `DEBUG` | `False` |
| `SECRET_KEY` | Унікальний секретний ключ Django |
| `SITE_DOMAIN` | Справжній домен, наприклад `shop.example.com` |
| `ALLOWED_HOSTS` | Домен сайту та `localhost` для внутрішніх healthcheck-запитів |
| `CSRF_TRUSTED_ORIGINS` | HTTPS-адреса сайту, наприклад `https://shop.example.com` |
| `DATABASE_URL` | URL підключення до PostgreSQL із хостом `db` |
| `POSTGRES_USER` | Користувач PostgreSQL |
| `POSTGRES_PASSWORD` | Надійний пароль PostgreSQL |
| `POSTGRES_DB` | Назва бази даних |
| `EMAIL_HOST` | `smtp.resend.com`, якщо використовуєш Resend |
| `EMAIL_PORT` | `587` |
| `EMAIL_HOST_USER` | `resend` для Resend SMTP |
| `EMAIL_HOST_PASSWORD` | API-ключ Resend |
| `DEFAULT_FROM_EMAIL` | Адреса відправника, дозволена для використання в Resend |
| `NOVA_POSHTA_API_KEY` | API-ключ Нової Пошти |

Згенерувати секретний ключ можна командою:

```bash
python -c "import secrets; print(secrets.token_urlsafe(50))"
```

Скопіюй результат у `SECRET_KEY`. Не використовуй `change-me` у production.

Параметри `DATABASE_URL` повинні відповідати `POSTGRES_USER`, `POSTGRES_PASSWORD` і `POSTGRES_DB`. У URL пароль зі спеціальними символами потрібно коректно кодувати.

Якщо `EMAIL_HOST` порожній, Django використовуватиме консольний бекенд: листи виводитимуться в журнали, а не надсилатимуться реально.

**Не публікуй `.env.production` і не зберігай його в репозиторії.**

### 4. Запусти production

```bash
docker compose -f docker-compose.prod.yml up -d --build
```

Створи адміністратора:

```bash
docker compose -f docker-compose.prod.yml exec web python manage.py createsuperuser
```

Переглянь журнали застосунку:

```bash
docker compose -f docker-compose.prod.yml logs -f web
```

Переглянь журнали Caddy:

```bash
docker compose -f docker-compose.prod.yml logs -f caddy
```

За правильної конфігурації домену та мережевого доступу Caddy автоматично отримає й оновлюватиме TLS-сертифікат.

### 5. Оновлення

Після отримання змін із репозиторію:

```bash
git pull
docker compose -f docker-compose.prod.yml up -d --build
```

Контейнер `web` застосовує міграції під час старту.

Перед оновленням production рекомендується зробити резервну копію бази даних і перевірити зміни міграцій.

## REST API

Документація API доступна за адресами `/api/docs/` (Swagger) і `/api/redoc/` (ReDoc).

| Ресурс | Методи та поведінка |
|---|---|
| `/api/products/`, `/api/products/<id>/` | GET — перегляд товарів; POST/PUT/PATCH/DELETE — керування товарами для staff |
| `/api/products/<id>/reviews/` | GET — перегляд відгуків; POST — створення відгуку за умовами покупки та обмеженням на повторний відгук |
| `/api/categories/` | GET — перегляд категорій |
| `/api/cart/` | GET — перегляд; POST — додавання товару; PATCH — зміна кількості; DELETE — видалення товару через `?product=<id>` або очищення кошика |
| `/api/orders/` | GET — власні замовлення, за потреби з фільтром `?status=`; POST — створення замовлення з кошика |
| `/api/orders/<id>/` | PATCH — зміна статусу для staff; DELETE — скасування замовлення відповідно до правил застосунку |
| `/api/users/register/` | Реєстрація |
| `/api/users/login/` | Отримання JWT-токенів |
| `/api/users/token/refresh/` | Оновлення access-токена |
| `/api/users/me/` | Отримання даних поточного користувача |

Фільтри товарів: `search`, `category`, `min_price`, `max_price`, `in_stock`, `ordering`.

Кошик в API використовує ту саму сесію, що й кошик на сайті. Під час роботи через Swagger або Postman потрібно зберігати й передавати сесійні cookie. Для захищених API-ендпоїнтів також може знадобитися JWT-токен.

## Безпека та секрети

- Не коміть `.env.local`, `.env.dev` або `.env.production`.
- Не використовуй демонстраційні паролі й `SECRET_KEY` у production.
- Налаштовуй `ALLOWED_HOSTS` і `CSRF_TRUSTED_ORIGINS` відповідно до реального домену.
- Регулярно створюй резервні копії PostgreSQL і перевіряй можливість їх відновлення.
