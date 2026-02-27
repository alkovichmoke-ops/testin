# Веб-приложение с регистрацией (Node.js + PostgreSQL)

Простое веб-приложение с функционалом регистрации, входа и выхода пользователя.

## Стек технологий

- **Бэкенд**: Node.js + Express
- **База данных**: PostgreSQL
- **Аутентификация**: express-session + bcrypt
- **Веб-сервер**: nginx (reverse proxy)

## Требования

- Node.js >= 16.x
- PostgreSQL >= 12
- nginx

---

## Развёртывание

### Шаг 1: Установка зависимостей

#### Установка Node.js (если не установлен)

```bash
# Ubuntu/Debian
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs

# Проверка версии
node --version
npm --version
```

#### Установка PostgreSQL

```bash
# Ubuntu/Debian
sudo apt-get update
sudo apt-get install -y postgresql postgresql-contrib

# Проверка статуса
sudo systemctl status postgresql
```

#### Установка nginx

```bash
sudo apt-get install -y nginx
sudo systemctl enable nginx
```

---

### Шаг 2: Настройка базы данных PostgreSQL

```bash
# Вход в PostgreSQL от имени пользователя postgres
sudo -u postgres psql
```

```sql
-- Создание базы данных
CREATE DATABASE registration_db;

-- Создание пользователя (опционально, если не хотите использовать postgres)
CREATE USER app_user WITH PASSWORD 'your_secure_password';
GRANT ALL PRIVILEGES ON DATABASE registration_db TO app_user;

-- Выход из psql
\q
```

Инициализация схемы БД:

```bash
# Подключение к БД и выполнение скрипта
sudo -u postgres psql -d registration_db -f /path/to/project/sql/init.sql
```

---

### Шаг 3: Установка зависимостей проекта

```bash
cd /home/mok/Documents/testin

# Установка npm-пакетов
npm install --production
```

---

### Шаг 4: Настройка переменных окружения

Создайте файл `.env` в корне проекта:

```bash
cp .env.example .env
nano .env
```

Пример содержимого `.env`:

```env
PORT=3000
DB_HOST=localhost
DB_PORT=5432
DB_NAME=registration_db
DB_USER=postgres
DB_PASSWORD=your_postgres_password
SESSION_SECRET=your-very-secret-key-change-this-in-production
```

> **Важно**: В production измените `SESSION_SECRET` на случайную строку!

---

### Шаг 5: Настройка systemd для автозапуска

Создайте файл службы systemd:

```bash
sudo nano /etc/systemd/system/registration-app.service
```

Содержимое:

```ini
[Unit]
Description=Registration App (Node.js)
After=network.target postgresql.service

[Service]
Type=simple
User=mok
WorkingDirectory=/home/mok/Documents/testin
ExecStart=/usr/bin/node /home/mok/Documents/testin/server.js
Restart=on-failure
RestartSec=10
Environment=NODE_ENV=production

# Логирование
StandardOutput=journal
StandardError=journal
SyslogIdentifier=registration-app

[Install]
WantedBy=multi-user.target
```

Активация и запуск:

```bash
# Перезагрузка systemd
sudo systemctl daemon-reload

# Включение автозапуска
sudo systemctl enable registration-app

# Запуск службы
sudo systemctl start registration-app

# Проверка статуса
sudo systemctl status registration-app

# Просмотр логов
sudo journalctl -u registration-app -f
```

---

### Шаг 6: Настройка nginx (reverse proxy)

Создайте конфигурационный файл nginx:

```bash
sudo nano /etc/nginx/sites-available/registration-app
```

Содержимое:

```nginx
server {
    listen 80;
    server_name your-domain.com;  # Замените на ваш домен или IP

    # Логирование
    access_log /var/log/nginx/registration-app-access.log;
    error_log /var/log/nginx/registration-app-error.log;

    # Проксирование на Node.js приложение
    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }

    # Статические файлы (опционально, nginx может отдавать их напрямую)
    location /static/ {
        alias /home/mok/Documents/testin/public/;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
}
```

Активация конфигурации:

```bash
# Создание симлинка
sudo ln -s /etc/nginx/sites-available/registration-app /etc/nginx/sites-enabled/

# Удаление дефолтной конфигурации (если конфликтует)
sudo rm /etc/nginx/sites-enabled/default

# Проверка конфигурации nginx
sudo nginx -t

# Перезагрузка nginx
sudo systemctl reload nginx
```

---

### Шаг 7: Настройка брандмауэра (опционально)

```bash
# Разрешить HTTP (порт 80)
sudo ufw allow 'Nginx Full'

# Проверка статуса
sudo ufw status
```

---

## Проверка работы

1. Откройте браузер и перейдите на `http://your-domain.com` или `http://your-server-ip`
2. Нажмите «Зарегистрироваться»
3. Заполните форму регистрации
4. После регистрации вы будете перенаправлены в личный кабинет
5. Проверьте выход и повторный вход

---

## Логи

- **Приложение**: `sudo journalctl -u registration-app -f`
- **nginx access**: `sudo tail -f /var/log/nginx/registration-app-access.log`
- **nginx error**: `sudo tail -f /var/log/nginx/registration-app-error.log`
- **PostgreSQL**: `sudo tail -f /var/log/postgresql/postgresql-*.log`

---

## Безопасность

### Рекомендации для production:

1. **HTTPS**: Настройте SSL-сертификат через Let's Encrypt:
   ```bash
   sudo apt-get install certbot python3-certbot-nginx
   sudo certbot --nginx -d your-domain.com
   ```

2. **Пароль сессии**: Используйте случайную строку для `SESSION_SECRET`:
   ```bash
   node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
   ```

3. **Ограничение прав БД**: Создайте отдельного пользователя БД с минимальными правами.

4. **Брандмауэр**: Закройте порт 3000 для внешнего доступа:
   ```bash
   sudo ufw deny 3000
   ```

---

## Структура проекта

```
/home/mok/Documents/testin/
├── server.js              # Основной файл приложения
├── package.json           # Зависимости npm
├── .env                   # Переменные окружения (не в git!)
├── .env.example           # Пример переменных окружения
├── sql/
│   └── init.sql          # Скрипт инициализации БД
└── public/
    ├── login.html        # Страница входа
    ├── register.html     # Страница регистрации
    └── dashboard.html    # Личный кабинет
```

---

## API

| Метод | Эндпоинт | Описание |
|-------|----------|----------|
| POST | `/api/register` | Регистрация пользователя |
| POST | `/api/login` | Вход |
| POST | `/api/logout` | Выход |
| GET | `/api/me` | Получение данных текущего пользователя |

### Пример запроса на регистрацию

```json
POST /api/register
Content-Type: application/json

{
  "username": "john",
  "email": "john@example.com",
  "password": "securepassword123"
}
```

### Пример ответа

```json
{
  "success": true,
  "user": {
    "id": 1,
    "username": "john",
    "email": "john@example.com"
  }
}
```
