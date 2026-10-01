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

