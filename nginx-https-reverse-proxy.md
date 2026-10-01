# Настройка HTTPS на Nginx с reverse proxy для backend-приложения

## Обзор

Nginx принимает HTTPS-трафик на порту 443, расшифровывает TLS и
проксирует запросы на backend-приложение, слушающее локально на
`127.0.0.1:8080`.

```mermaid
flowchart LR
    A[Клиент] -- "HTTPS :443" --> B[Nginx]
    B -- "HTTP :8080" --> C[Backend]
```

## Prerequisites

- Linux-сервер (примеры для Ubuntu/Debian), доступ `sudo`.
- Backend уже запущен и слушает `127.0.0.1:8080`.
- Порт `443/tcp` (и `80/tcp` для редиректа) открыт в файрволе.
- Self-signed сертификат подходит для теста/внутреннего использования;
  для production — сертификат от доверенного CA (например, Let's Encrypt).

## Шаг 1. Установка Nginx

```bash
sudo apt update && sudo apt install -y nginx
sudo systemctl enable --now nginx
```

[Screenshot: вывод `systemctl status nginx` — `active (running)`]

## Шаг 2. Генерация self-signed сертификата

```bash
sudo mkdir -p /etc/nginx/ssl
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/selfsigned.key \
  -out /etc/nginx/ssl/selfsigned.crt \
  -subj "/CN=example.local"
sudo chmod 600 /etc/nginx/ssl/selfsigned.key
```

[Screenshot: содержимое `/etc/nginx/ssl/` с файлами `.crt` и `.key`]

## Шаг 3. Server block с SSL и proxy_pass

`/etc/nginx/sites-available/backend-app.conf`:

```nginx
server {
    listen 80;
    server_name example.local;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name example.local;

    ssl_certificate     /etc/nginx/ssl/selfsigned.crt;
    ssl_certificate_key /etc/nginx/ssl/selfsigned.key;

    location / {
        proxy_pass         http://127.0.0.1:8080;
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-Proto $scheme;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/backend-app.conf /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
```

[Screenshot: файл конфигурации в редакторе `nano`]

## Шаг 4. Проверка и запуск

```bash
sudo nginx -t
sudo systemctl reload nginx
curl -kI https://example.local
```

[Screenshot: `nginx -t` → `syntax is ok`, `curl` → `200 OK`]

## Параметры конфигурации

| Параметр | Назначение |
|---|---|
| `listen 443 ssl` | Включает приём HTTPS-соединений |
| `server_name` | Домен, для которого действует блок |
| `ssl_certificate` / `ssl_certificate_key` | Пути к сертификату и ключу |
| `proxy_pass` | Адрес backend-приложения |
| `proxy_set_header Host` | Передаёт исходный `Host` на backend |
| `proxy_set_header X-Real-IP` | Передаёт реальный IP клиента |
| `proxy_set_header X-Forwarded-Proto` | Сообщает backend, что запрос был по HTTPS |

## Troubleshooting

**1. `Address already in use` на 443**
Порт занят другим процессом. `sudo ss -tulpn | grep :443`, остановите
конфликтующую службу (например, Apache) и перезапустите Nginx.

**2. `502 Bad Gateway`**
Backend недоступен. Проверьте `ss -tulpn | grep 8080`,
`curl http://127.0.0.1:8080` и лог `/var/log/nginx/error.log`.

**3. Предупреждение о недоверенном сертификате**
Ожидаемо для self-signed. Для теста используйте `curl -k`; для
production подключите сертификат от доверенного CA.