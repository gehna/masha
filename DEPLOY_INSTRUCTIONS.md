## Деплой сайта www.maria-legkodukhova.ru на Ubuntu 22.04
Работаем: подключение по SSH под `root`, затем переключаемся на `ubuntu`. Файлы правим через `nano`.

### 1) Логин под root и создание пользователя ubuntu
```bash
ssh root@<SERVER_IP>
adduser ubuntu
usermod -aG sudo ubuntu
echo "ubuntu ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/ubuntu
chmod 440 /etc/sudoers.d/ubuntu
```

### 2) Переключение на ubuntu для всех дальнейших действий
```bash
su - ubuntu
```
(В следующих сессиях: `ssh root@<SERVER_IP>`, затем `su - ubuntu`).

### 3) Базовая подготовка системы (под пользователем ubuntu)
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y nginx certbot python3-certbot-nginx ufw git nano
```

### 4) Клонирование репозитория сайта
```bash
cd /var/www
git clone https://github.com/gehna/masha.git maria-legkodukhova.ru
sudo chown -R ubuntu:ubuntu /var/www/maria-legkodukhova.ru
```
Если репозиторий обновляется, для деплоя обновлений:
```bash
cd /var/www/maria-legkodukhova.ru
git pull
```

### 5) Брандмауэр (ufw)
```bash
sudo ufw allow OpenSSH
sudo ufw allow "Nginx Full"
sudo ufw --force enable
sudo ufw status
```

### 6) DNS
- В панели домена создайте/проверьте:
  - `A` для `www.maria-legkodukhova.ru` → публичный IP сервера.
  - (Опц.) `A` для `maria-legkodukhova.ru` → тот же IP.
- Подождите применения: `dig www.maria-legkodukhova.ru` должен вернуть ваш IP.

### 7) Конфиг nginx (редактируем nano)
```bash
sudo nano /etc/nginx/sites-available/maria-legkodukhova.ru
```
Вставьте:
```
server {
    listen 80;
    listen [::]:80;
    server_name www.maria-legkodukhova.ru maria-legkodukhova.ru;

    root /var/www/maria-legkodukhova.ru;
    index index.html;

    # Если SPA:
    # location / {
    #     try_files $uri /index.html;
    # }

    # Если нужен прокси на бэкенд:
    # location /api/ {
    #     proxy_pass http://127.0.0.1:3000;
    #     proxy_set_header Host $host;
    #     proxy_set_header X-Real-IP $remote_addr;
    #     proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    #     proxy_set_header X-Forwarded-Proto $scheme;
    # }
}
```
Сохраните (Ctrl+O, Enter, Ctrl+X).

Активируйте и перезагрузите nginx:
```bash
sudo ln -s /etc/nginx/sites-available/maria-legkodukhova.ru /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

### 8) HTTPS (Let’s Encrypt)
Убедитесь, что DNS уже указывает на сервер.
```bash
sudo certbot --nginx -d www.maria-legkodukhova.ru -d maria-legkodukhova.ru --agree-tos -m gehna@yandex.ru --redirect
```
Проверка автопродления:
```bash
sudo systemctl status certbot.timer
sudo certbot renew --dry-run
```

### 9) (Опционально) Бэкенд как systemd-сервис
Пример для Node (порт 3000):
```bash
sudo nano /etc/systemd/system/app.service
```
Вставьте:
```
[Unit]
Description=App service
After=network.target

[Service]
User=ubuntu
WorkingDirectory=/var/www/maria-legkodukhova.ru
ExecStart=/usr/bin/node server.js
Restart=always
Environment=NODE_ENV=production
# Добавьте нужные переменные окружения

[Install]
WantedBy=multi-user.target
```
Примените:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now app.service
sudo systemctl status app.service
```
В nginx раскомментируйте блок `location /api/` и перезагрузите nginx.

### 10) Проверки
```bash
curl -I http://www.maria-legkodukhova.ru
curl -I https://www.maria-legkodukhova.ru
```
В браузере: сертификат валиден, редирект на HTTPS работает.

### 11) Обслуживание
- Обновления: `sudo apt update && sudo apt upgrade -y` периодически.
- Логи: `sudo journalctl -u nginx -f`, `sudo journalctl -u app.service -f`, `/var/log/nginx/error.log`.
- Для теста до DNS: в своём `/etc/hosts` добавить строку `<SERVER_IP> www.maria-legkodukhova.ru` (и при необходимости `maria-legkodukhova.ru`).
