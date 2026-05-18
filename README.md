# Настройка VLESS + xHTTP через CDN (Timeweb Cloud)

Репозиторий содержит пошаговое руководство по развертыванию и оптимизации отказоустойчивого прокси-соединения с использованием протокола **VLESS** и транспортного протокола **xHTTP** (пакетный режим `packet-up`), скрытого за распределенной сетью доставки контента (CDN).

## 📊 Схема архитектуры

```text
[ Клиент ] 
    │
    ▼ (TLS / h2, маскировка под обычный трафик)
[ Timeweb CDN ]
    │
    ▼ (HTTPS / Порт 443)
[ Nginx Reverse Proxy ] (Авторизация сертификата Let's Encrypt)
    │
    ▼ (HTTP / Локальный порт 10085)
[ Xray / Remnawave ]
```
---
## 1. Настройка CDN
1. Заходим на сайт: [Timeweb Cloud](https://timeweb.cloud)
2. Переходим в раздел **CDN**.
3. Нажимаем на кнопку **«Добавить»**.
4. Выбираем тип: **Домен**.
5. Вписываем домен, который направлен на ваш Origin-сервер.

> ⚠️ **Важно:** Обязательно отметьте галочкой пункт **«Использовать HTTPS соединение для источника»**.

6. Заходим в управление созданным CDN и выставляем параметры по таблице ниже.

### Итоговые настройки CDN:

| Параметр | Значение |
| :--- | :--- |
| Источник контента | Домен |
| Порт источника | 443 |
| HTTPS к источнику | Включено |
| Домены раздачи | Технический CDN-домен + свой CNAME при необходимости |
| CDN-кэширование | Выключено |
| Кэширование в браузере | Выключено |
| Учитывать query string | Включено, режим: Все |
| Всегда онлайн | Выключено |
| Редирект HTTP -> HTTPS | Выключено |
| SSL-сертификат CDN | Не трогать, если клиент использует технический CDN-домен |
| Secure token | Выключено |
| AWS-авторизация | Выключено |
| HTTP/3 | Выключено |
| Gzip | Выключено |
| Ускорение больших файлов | Выключено |
| Заголовки запросов | Выключено; custom Host не добавлять |
| Дополнительные HTTP-методы | Включить POST, PUT |

🛈 **Важное примечание по CDN:**
CDN-кэширование и кэширование в браузере должны быть строго выключены. Query string нужно учитывать полностью. Метод POST должен быть обязательно разрешен, иначе пакетный режим xHTTP не сможет отправлять аплинки.

---
## 2. Настройка Origin-сервера (Nginx)

### • Устанавливаем Nginx, Certbot и открываем конфиг:
```bash
apt update && apt install -y nginx certbot && nano /etc/nginx/sites-available/ВАШ_ДОМЕН
```

### • Вставляем
```bash
server {
    listen 443 ssl http2;
    server_name ВАШ_ДОМЕН;

    ssl_certificate     /etc/letsencrypt/live/ВАШ_ДОМЕН/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/ВАШ_ДОМЕН/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:10085;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_buffering off;
        proxy_request_buffering off;
        proxy_cache off;
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
        client_max_body_size 0;
    }
}
```
Сохраняем и выходим ctrl + o, enter, ctrl + x

### • Выпуск SSL и запуск веб-сервера
```bash
certbot certonly --standalone -d ВАШ_ДОМЕН
```

### • Активируем конфиг, проверяем и перезапускаем Nginx
```bash
ln -sf /etc/nginx/sites-available/ВАШ_ДОМЕН /etc/nginx/sites-enabled/ВАШ_ДОМЕН && nginx -t && systemctl enable nginx && systemctl restart nginx
```

---
## 3. Оптимизация сетевых буферов (Обязательно)
### • Включение bbr
```bash
echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf && echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf && sysctl -p
```

### • Открываем файл
```bash
nano /etc/sysctl.conf
```

### • Вставляем в самом конце файлв
```bash
net.local.port_range = 1024 65535
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_max_syn_backlog = 4096
net.core.somaxconn = 4096
```
### • Примените внесенные изменения без перезагрузки сервера:
```bash
sysctl -p
```
---
## 4. Настройка Remnawave / Xray

### • Конфигурация входящего соединения (Inbound)

В панели управления Remnawave cоздайте новый профиль, удалите содержимое и вставьте данный конфиг:

```
{
  "log": {
    "loglevel": "warning"
  },
  "dns": {},
  "inbounds": [
    {
      "tag": "CDN_XHTTP_RAWINET",
      "port": 10085,
      "listen": "127.0.0.1",
      "protocol": "vless",
      "settings": {
        "clients": [],
        "decryption": "none"
      },
      "sniffing": {
        "enabled": true,
        "destOverride": [
          "http",
          "tls",
          "quic"
        ]
      },
      "streamSettings": {
        "network": "xhttp",
        "security": "none",
        "xhttpSettings": {
          "mode": "packet-up",
          "path": "/",
          "extra": {
            "path": "/",
            "seqKey": "page",
            "sessionKey": "X-Auth-Token",
            "xPaddingKey": "_dc",
            "seqPlacement": "query",
            "xPaddingHeader": "X-Cache",
            "xPaddingMethod": "tokenish",
            "sessionPlacement": "header",
            "uplinkHTTPMethod": "POST",
            "xPaddingObfsMode": true,
            "xPaddingPlacement": "queryInHeader"
          },
          "noSSEHeader": false
        }
      }
    }
  ],
  "outbounds": [
    {
      "tag": "DIRECT",
      "protocol": "freedom"
    },
    {
      "tag": "BLOCK",
      "protocol": "blackhole"
    }
  ],
  "routing": {
    "rules": []
  }
}
```

### • Параметры Хоста

Заполните параметры подключения со стороны клиента. Замените значение CDN_DOMAIN на ваш технический адрес CDN от Timeweb (например, xxxxxxx.cdn.twcstorage.ru) или на выделенный собственный CNAME-домен:

```
"address": "CDN_DOMAIN",
"port": 443,
"host": "CDN_DOMAIN",
"sni": "CDN_DOMAIN",
"securityLayer": "TLS",
"alpn": "h2,http/1.1",
"fingerprint": "chrome"
```
> 🛈 **Обратите внимание:** Для успешного прохождения рукопожатия TLS, значение домена в полях `address`, `host` и `sni` должно быть абсолютно идентичным.

### • Дополнительные параметры xHTTP (extra params)

Рабочий набор параметров:

```
{
  "path": "/",
  "seqKey": "page",
  "sessionKey": "X-Auth-Token",
  "xPaddingKey": "_dc",
  "seqPlacement": "query",
  "xPaddingHeader": "X-Cache",
  "xPaddingMethod": "tokenish",
  "sessionPlacement": "header",
  "uplinkHTTPMethod": "POST",
  "xPaddingObfsMode": true,
  "xPaddingPlacement": "queryInHeader"
}
```
---

## 📬 Обратная связь

Если у вас возникли вопросы по настройке, конфигурации или вы нашли ошибку, пишите в Telegram:
👉 [@Rawi_v](https://t.me/Rawi_v)

## ☕ Поддержать проект (Donations)

Если эта инструкция сэкономила ваше время, помогла разобраться в технологии или защитить ваш трафик, вы можете отблагодарить автора материально:

* **USDT (TRC-20 / Tron):**
    ```text
    TNH2JJV7tvsNfJ2dkiNi7wS5qhzjz12YT6
    ```
* **USDT (BEP-20 / BNB Smart Chain):**
    ```text
    0x735b133ADA406f24960e7918DF6F5f87232cFD0d
    ```
