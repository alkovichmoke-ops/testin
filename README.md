# Веб-приложение с регистрацией (Node.js + PostgreSQL)

Простое веб-приложение с функционалом регистрации, входа и выхода пользователя.

## Стек технологий

- **Бэкенд**: Node.js + Express
- **База данных**: PostgreSQL
- **Аутентификация**: express-session + bcrypt
- **Веб-сервер**: nginx (reverse proxy)

## Требования

- Node.js >= 22.x (LTS)
- PostgreSQL >= 12
- nginx

---

## Быстрый старт на Ubuntu 24.04 Server

### 1. Обновление системы и установка зависимостей

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl git build-essential
```

### 2. Установка Node.js 22.x

```bash
# Добавление репозитория NodeSource
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs

# Проверка
node --version  # должно быть v22.x
npm --version
```

### 3. Установка PostgreSQL

```bash
sudo apt install -y postgresql postgresql-contrib
sudo systemctl enable postgresql
sudo systemctl start postgresql
```

### 4. Клонирование/копирование проекта

```bash
# Если используете git
git clone <repository-url> /opt/registration-app
# Или скопируйте файлы проекта в /opt/registration-app

cd /opt/registration-app
```

### 5. Настройка базы данных

```bash
# Создание БД
sudo -u postgres psql -c "CREATE DATABASE registration_db;"

# Инициализация схемы
sudo -u postgres psql -d registration_db -f sql/init.sql
```

### 6. Установка зависимостей npm

```bash
npm install --omit=dev
```

> Если зависает — проверьте DNS и доступ к registry.npmjs.org:
> ```bash
> ping registry.npmjs.org
> # Или используйте зеркало:
> npm config set registry https://registry.npmmirror.com
> npm install --omit=dev
> ```

### 7. Настройка .env

```bash
cp .env.example .env
nano .env
```

### 8. Запуск

```bash
# Тестовый запуск
npm start

# Или через systemd (см. ниже)
```

---

## Полная инструкция по развёртыванию

### Настройка базы данных PostgreSQL

```bash
# Вход в PostgreSQL
sudo -u postgres psql
```

```sql
-- Создание БД
CREATE DATABASE registration_db;

-- Опционально: создание отдельного пользователя
CREATE USER app_user WITH PASSWORD 'your_secure_password';
GRANT ALL PRIVILEGES ON DATABASE registration_db TO app_user;
\q
```

Инициализация схемы:

```bash
sudo -u postgres psql -d registration_db -f sql/init.sql
```

---

### Настройка переменных окружения

```bash
cp .env.example .env
nano .env
```

Пример `.env`:

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

### Настройка systemd для автозапуска

Создайте файл службы:

```bash
sudo nano /etc/systemd/system/registration-app.service
```

Содержимое (для Ubuntu 24.04):

```ini
[Unit]
Description=Registration App (Node.js)
After=network.target postgresql.service

[Service]
Type=simple
# Создайте пользователя для приложения или используйте существующего
User=www-data
Group=www-data
WorkingDirectory=/opt/registration-app
ExecStart=/usr/bin/node /opt/registration-app/server.js
Restart=on-failure
RestartSec=10
Environment=NODE_ENV=production
Environment=PATH=/usr/bin:/usr/local/bin

# Логирование
StandardOutput=journal
StandardError=journal
SyslogIdentifier=registration-app

# Безопасность
NoNewPrivileges=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

Активация:

```bash
# Перезагрузка systemd
sudo systemctl daemon-reload

# Включение автозапуска
sudo systemctl enable registration-app

# Запуск
sudo systemctl start registration-app

# Проверка статуса
sudo systemctl status registration-app

# Просмотр логов
sudo journalctl -u registration-app -f
```

---

### Настройка nginx (reverse proxy)

Установка nginx:

```bash
sudo apt install -y nginx
sudo systemctl enable nginx
```

Создайте конфигурационный файл:

```bash
sudo nano /etc/nginx/sites-available/registration-app
```

Содержимое (для Ubuntu 24.04):

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

    # Статические файлы
    location /static/ {
        alias /opt/registration-app/public/;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
}
```

Активация:

```bash
# Создание симлинка
sudo ln -s /etc/nginx/sites-available/registration-app /etc/nginx/sites-enabled/

# Удаление дефолтной конфигурации (если есть)
sudo rm -f /etc/nginx/sites-enabled/default

# Проверка конфигурации
sudo nginx -t

# Перезагрузка nginx
sudo systemctl reload nginx
```

> **Примечание для Ubuntu 24.04**: В новых версиях nginx может использовать `/etc/nginx/conf.d/` вместо `sites-available/sites-enabled`.  
> Альтернативный вариант:
> ```bash
> sudo cp /etc/nginx/sites-available/registration-app /etc/nginx/conf.d/registration-app.conf
> ```

---

### Настройка брандмауэра (UFW)

```bash
# Разрешить HTTP и HTTPS
sudo ufw allow 'Nginx Full'

# Разрешить SSH (если нужно)
sudo ufw allow 'OpenSSH'

# Включить брандмауэр
sudo ufw enable

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
   # Установка Certbot
   sudo apt install -y certbot python3-certbot-nginx
   
   # Получение сертификата
   sudo certbot --nginx -d your-domain.com
   
   # Автообновление (добавлено в cron автоматически)
   sudo certbot renew --dry-run
   ```

2. **Пароль сессии**: Сгенерируйте случайную строку:
   ```bash
   node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
   ```

3. **Ограничение прав БД**: Создайте отдельного пользователя с минимальными правами:
   ```sql
   CREATE USER app_user WITH PASSWORD 'secure_password';
   GRANT SELECT, INSERT, UPDATE ON users TO app_user;
   ```

4. **Брандмауэр**: Закройте порт 3000:
   ```bash
   sudo ufw deny 3000
   ```

5. **Пользователь для приложения**: Создайте отдельного пользователя:
   ```bash
   sudo useradd -r -s /bin/false registration-app
   sudo chown -R registration-app:registration-app /opt/registration-app
   ```

---

## Структура проекта

```
/opt/registration-app/
├── server.js              # Основной файл приложения
├── package.json           # Зависимости npm
├── .env                   # Переменные окружения (не в git!)
├── .env.example           # Пример переменных окружения
├── .gitignore
├── README.md              # Эта инструкция
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

---

## Решение проблем

### npm install зависает

1. Проверьте подключение к интернету:
   ```bash
   ping registry.npmjs.org
   ```

2. Используйте зеркало (если в РФ):
   ```bash
   npm config set registry https://registry.npmmirror.com
   npm install --omit=dev
   ```

3. Очистите кэш npm:
   ```bash
   npm cache clean --force
   npm install
   ```

### Ошибка подключения к PostgreSQL

1. Проверьте статус:
   ```bash
   sudo systemctl status postgresql
   ```

2. Проверьте `pg_hba.conf`:
   ```bash
   sudo nano /etc/postgresql/*/main/pg_hba.conf
   # Убедитесь, что есть: local all postgres peer
   ```

3. Перезапустите PostgreSQL:
   ```bash
   sudo systemctl restart postgresql
   ```

### Приложение не запускается

1. Проверьте логи:
   ```bash
   sudo journalctl -u registration-app -n 50
   ```

2. Проверьте `.env`:
   ```bash
   cat /opt/registration-app/.env
   ```

3. Запустите вручную для отладки:
   ```bash
   cd /opt/registration-app
   node server.js
   ```
