# vidos-nginx-xray
Сюда залил все что было в видосе + конкретней через иишку прогнал 
Скрипт внизу


```nginx
# default_server для обработки мусорных запросов по HTTP (Drop)
server {
    listen 80 default_server;
    server_name _;
    include tor_block.conf;
    error_page 400 =444 /;
    error_page 403 =444 /;
    error_page 497 =444 /;
    location / {
        return 444;
    }
    return 444;
}

# default_server для обработки мусорных запросов по HTTPS с фейковым сертификатом (Drop)
server {
    listen 443 ssl default_server;
    server_name _;
    include tor_block.conf;

    ssl_certificate /etc/ssl/certs/fake.crt;
    ssl_certificate_key /etc/ssl/private/fake.key;
    error_page 400 =444 /;
    error_page 403 =444 /;
    error_page 497 =444 /;
    location / {
        return 444;
    }
    return 444;
}

# Редирект HTTP -> HTTPS для основного домена
server {
    listen 80;
    server_name yourdomain.com;
    include tor_block.conf;
    error_page 400 =402 /;
    error_page 403 =444 @drop;
    error_page 497 =402 /;
    
    location @drop {
        return 444;
    }

    location / {
        return 402;
    }

    error_page 402 =301 /301;

    location /301 {
        root /var/web/fake80;
        try_files /301.html =301;
        add_header Location "https://$host$request_uri" always;
    }
}

# Редирект HTTP -> HTTPS для CDN-поддомена
server {
    listen 80;
    server_name cdn.yourdomain.com;
    include tor_block.conf;
    error_page 400 =402 /;
    error_page 403 =444 @drop;
    error_page 497 =402 /;
    
    location @drop {
        return 444;
    }

    location / {
        return 402;
    }

    error_page 402 =301 /301;

    location /301 {
        root /var/web/fake80;
        try_files /3011.html =301;
        add_header Location "https://$host$request_uri" always;
    }
}

# Основной сервер (yourdomain.com)
server {
    listen 443 ssl;
    http2 on;
    server_name yourdomain.com;
    include tor_block.conf;
    
    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem]; 
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem; 
    http2_max_concurrent_streams 512;

    error_page 403 =444 @drop;
    location @drop {
        return 444;
    }
    
    # Кастомная обработка ошибок
    error_page 402 403 404 405 408 412 413 414 415 416 500 501 502 503 504 =404 /404.html;
    error_page 400 /400.html;
    
    location = /404.html {
        root /var/www/html;
        internal;
    }
    location = /401.html {
        root /var/www/html;
        internal;
    }
    location = /400.html {
        root /var/www/html;
        internal;
    }

    # Блокировка известных сканеров и ботов
    if ($http_user_agent ~* (netcrawler|MegaIndex|ZmEu|python|nikto|dirbuster|sqlmap|censys|shodan)) {
        return 444;
    }
    
    access_log /var/log/nginx/yourdomain.com.log custom;
    error_log /var/log/nginx/yourdomain.com.log;
   
    # Security Headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    
    location = /favicon.ico {
        root /var/www/html;
        access_log off;
        log_not_found off;
    }
    
    location = /robots.txt {
        root /var/www/html;
        allow all;
        log_not_found off;
        access_log off;
    }
  
    # Главная страница фейкового сайта
    location / {
        root /var/idk;
        index index.html;
        try_files $uri $uri/ =404;
    }

    # Проксирование панели 
    location /your/path {
        auth_basic "Development Build v0.8.1 - Authorization Required";
        auth_basic_user_file /etc/nginx/.htpasswd;
        proxy_pass http://127.0.0.1:портпанели;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_redirect off;
        proxy_intercept_errors on;
        proxy_buffering off;
        proxy_request_buffering off;
    }

    # ws ssh
    location /your/path2 {
        if ($http_upgrade != "websocket") {
            return 404;
        }
        proxy_pass http://127.0.0.1:8022;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        proxy_read_timeout 7d;
        proxy_send_timeout 7d;
        proxy_buffering off;
    }
}

# CDN Сервер (cdn.yourdomain.com)
server {
    listen 443 ssl;
    http2 on;
    server_name cdn.yourdomain.com;
    access_log /var/log/nginx/cdn_yourdomain.com.log custom;
    error_log /var/log/nginx/cdn_yourdomain.com.log;

    ssl_certificate /etc/letsencrypt/live/cdn.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/cdn.yourdomain.com/privkey.pem;

    include tor_block.conf;
    
    error_page 401 402 404 405 408 412 413 414 415 416 500 501 502 503 504 =404 /4044.html;
    error_page 400 /4000.html;
    
    location = /4044.html {
        root /var/www/html;
        internal;
    }
    error_page 403 /403.html;
    
    location = /403.html {
        root /var/www/html; 
        internal;           
    }
    location = /4000.html {
        root /var/www/cdn_public;
        internal;
    }

    location = / {
        return 403;
        proxy_intercept_errors on;
    }
    
    http2_max_concurrent_streams 512;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # Отдача статических CSS-стилей
    location /your/path.css {
        root /var/www/cdn_public;
        types {
            "text/css; charset=utf-8" css;
        }
        try_files /css/style.css =404;
        add_header Cache-Control "public, max-age=31536000, immutable";
        add_header Access-Control-Allow-Origin "*" always;
    }

    # VPN
    location ~ ^/your/path\.js(?<vpn_path>/.*)?$ {
        set $block_client "0";
        if ($is_vpn_client = "0") {
            set $block_client "1";
        }
        if ($is_secret_agent = "0") {
            set $block_client "1";
        }
        if ($block_client = "1") {
            rewrite ^/your/path\.js$ /render_fake_cdn_javascript last;
            return 404;
        }
        access_log off;
        error_log /dev/null crit;
        proxy_pass http://127.0.0.1:10000;
        proxy_http_version 1.1;
        
        proxy_set_header Connection "";
        chunked_transfer_encoding off;
        proxy_buffering off;
        proxy_request_buffering off;
        proxy_set_header X-Accel-Buffering "no";
        
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        proxy_read_timeout 7d;
        proxy_send_timeout 7d;
    }

    # Маршрут для отдачи фейкового JS неавторизованным запросам
    location /render_fake_cdn_javascript {
        internal;
        access_log /var/log/nginx/vpn_scanners_blocked.log custom;
        root /var/www/cdn_public;
        types {
            "application/javascript; charset=utf-8" js;
        }
        try_files /js/fake.js =404;
        add_header Cache-Control "public, max-age=31536000, immutable";
        add_header Access-Control-Allow-Origin "*" always;
    }
}

Активируйте конфиг командой:
Bash

ln -s /etc/nginx/sites-available/yourdomain.com /etc/nginx/sites-enabled/yourdomain.com

Заменяет стандартный конфиг. Включает маскировку под Apache, оптимизацию обработки соединений, а также парсинг реальных IP от Cloudflare.
Nginx

user www-data;
worker_processes auto;
worker_cpu_affinity auto;
pid /run/nginx.pid;
error_log /var/log/nginx/error.log;
include /etc/nginx/modules-enabled/*.conf;

events {
    worker_connections 4096;
    multi_accept on;
    use epoll;
}

http {
    # CF_REALIP_BEGIN
    real_ip_header CF-Connecting-IP;
    real_ip_recursive on;
    set_real_ip_from 103.21.244.0/22;
    set_real_ip_from 103.22.200.0/22;
    set_real_ip_from 103.31.4.0/22;
    set_real_ip_from 104.16.0.0/13;
    set_real_ip_from 104.24.0.0/14;
    set_real_ip_from 108.162.192.0/18;
    set_real_ip_from 131.0.72.0/22;
    set_real_ip_from 141.101.64.0/18;
    set_real_ip_from 162.158.0.0/15;
    set_real_ip_from 172.64.0.0/13;
    set_real_ip_from 173.245.48.0/20;
    set_real_ip_from 188.114.96.0/20;
    set_real_ip_from 190.93.240.0/20;
    set_real_ip_from 197.234.240.0/22;
    set_real_ip_from 198.41.128.0/17;
    # CF_REALIP_BEGIN

    log_format custom '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_range" "$sent_http_content_range"';
                      
    map_hash_bucket_size 256;
    map_hash_max_size 4096;

    # Валидация секретных ключей для доступа к CDN ресурсам
    map $http_user_agent $is_secret_agent {
        "ваши куки/user_agent" "1";
        default "0";
    }
    map $http_cookie $is_vpn_client {
        default "0";
        "ваши куки" "1";
    }

    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;

    types_hash_max_size 2048;
    server_tokens off;

    # Затираем следы Nginx и притворяемся сервером Apache
    more_clear_headers "Server" "ETag" "X-Powered-By";
    more_set_headers "Server: Apache/2.4.62 (Unix)";
    etag off;
    reset_timedout_connection on;

    client_body_buffer_size 512k;
    client_header_buffer_size 16k;
    client_max_body_size 0;
    large_client_header_buffers 4 16k;

    keepalive_timeout 300s;
    keepalive_requests 10000;
    client_body_timeout 60s;
    client_header_timeout 60s;
    send_timeout 60s;

    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;

    access_log /var/log/nginx/access.log;
    gzip on;

    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}

Для корректной работы маскировки создайте следующие файлы и папки на сервере:
Bash

# Файлы-заглушки для дефолтного сайта и страниц ошибок
~# ls /var/www/html/
400.html  401.html  403.html  4044.html  404.html  favicon.ico  index.html  robots.txt

# Файлы для псевдо-CDN
~# ls /var/www/cdn_public/
4000.html  css  js

# Файлы для редиректов по 80 порту
~# ls /var/web/fake80/
3011.html  301.html

Рекомендуемый robots.txt

Положите его в /var/www/html/robots.txt для запрета индексации системных директорий поисковиками:
Plaintext

User-agent: *
Allow: /
Disallow: /assets/config/
Disallow: /private/
Disallow: /admin/
Disallow: /administrator/
Disallow: /wp-admin/
Disallow: /login/
Disallow: /dashboard/
Disallow: /backup/
Disallow: /backups/
Disallow: /db/
Disallow: /database/
Disallow: /dump/
Disallow: /sql/
Disallow: /.env
Disallow: /config/
Disallow: /logs/
Disallow: /dev/
Disallow: /api/v1/internal/
Disallow: /hidden/

Пример Bash-скрипта запуска nfqws для десинхронизации пакетов DPI провайдера. Конфигурация десинхронизирует TLS-сессии и UDP-пакеты.
Bash

#!/bin/bash

# Очистка старых правил
sudo iptables -t mangle -F
sudo ip rule del fwmark 0x40 table 100 2>/dev/null
sudo ip route flush table 100 2>/dev/null

# Пропускаем трафик системных демонов
sudo iptables -t mangle -A OUTPUT -m owner --uid-owner nobody -j RETURN

# Заворачиваем трафик на HTTPS (443) и кастомные UDP-порты в очередь NFQUEUE
sudo iptables -t mangle -A OUTPUT -p tcp --dport 443 -j NFQUEUE --queue-num 300 --queue-bypass
sudo iptables -t mangle -A OUTPUT -p udp --dport 50000:65535 -j NFQUEUE --queue-num 300 --queue-bypass

cd zapret-master/binaries/my

# Запуск демона nfqws с кастомным фейк-SNI и дублированием сессий
sudo ./nfqws --qnum=300 --hostlist=hosts.txt \
--dpi-desync=fake,multisplit --dpi-desync-split-pos=2 --dpi-desync-split-seqovl=681 --dpi-desync-fooling=ts --dpi-desync-repeats=4 --dpi-desync-fake-tls-mod=rnd,dupsid,sni=www.google.com\
--dpi-desync-any-protocol=1 --dpi-desync-udplen-increment=2 \
--bind-fix4 --bind-fix6 \
--user=nobody &

Шаг A: Изоляция SSH на сервере

Сделайте так, чтобы демон SSH не висел открытым на внешнем IP-адресе.
Отредактируйте /etc/ssh/sshd_config:
Plaintext

Include /etc/ssh/sshd_config.d/*.conf

Port 22
ListenAddress 127.0.0.1

Перезапустите демон:
Bash

sudo systemctl restart ssh
sudo systemctl restart sshd

Шаг B: Запуск wstunnel в качестве systemd службы на VPS

Создайте сервис-файл /etc/systemd/system/wstunnel.service:
Ini, TOML

[Unit]
Description=wstunnel SSH over WebSocket Server
After=network.target

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/wstunnel server ws://127.0.0.1:8022 --restrict-to 127.0.0.1:22
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target

Примените настройки и запустите:
Bash

sudo systemctl daemon-reload
sudo systemctl enable wstunnel.service
sudo systemctl start wstunnel.service

Шаг C: Клиентский скрипт для подключения

Скрипт создает локальный сокет на порту 2222, пробрасывающий SSH-трафик сквозь WebSockets в HTTPS-сессию Nginx:
Bash

#!/bin/bash
./wstunnel client -L tcp://127.0.0.1:2222:127.0.0.1:22 --http-upgrade-path-prefix your/path wss://yourdomain.com > tunnel.log 2>&1 &

Для подключения к VPS теперь достаточно запустить: ssh user@127.0.0.1 -p 2222
Тонкие настройки xHTTP в v2rayN

Вставьте этот JSON в расширенные настройки транспорта xHTTP на клиенте:
JSON

{
  "headers": {
    "Ваш_Секретный_Заголовок": "куки_значение",
    "Другой_Заголовок": "еще_куки"
  },
  "mode": "auto",
  "scMaxEachPostBytes": "10000000",
  "scMinPostsIntervalMs": "10-200",
  "uplinkHTTPMethod": "PUT",
  "xPaddingBytes": "100-500",
  "xmux": {
    "cMaxReuseTimes": 0,
    "hKeepAlivePeriod": 0,
    "hMaxRequestTimes": "600-900",
    "hMaxReusableSecs": "1800-3000",
    "maxConnections": 6
  }
}

Рекомендуемые DNS настройки

    DNS для локальных (прямых) подключений: tls://1.1.1.1

    Удаленный (внутри туннеля) DNS: tls://8.8.8.8

    Bootstrap DNS: tls://1.1.1.1

Настройка Браузера (DoH)

Для обхода локальных утечек DNS включите во вкладке приватности браузера технологию DNS-over-HTTPS (DoH) и укажите адрес сервера:
Plaintext
https://dns.google/dns-query
Bash

# 1. Обновление ОС
sudo apt update && sudo apt upgrade -y

# 2. Установка веб-сервера и утилит для SSL
sudo apt install nginx certbot python3-certbot-nginx -y

# 3. Развертывание панели 3X-UI
bash <(curl -Ls https://raw.githubusercontent.com/mhsanaei/3x-ui/master/install.sh)

# 4. Выпуск Let's Encrypt SSL-сертификата
sudo systemctl enable nginx
sudo systemctl start nginx
sudo certbot --nginx -d yourdomain.com

# 5. Конфигурирование Nginx (см. Шаг 1-2)
sudo nano /etc/nginx/sites-available/default
sudo systemctl restart nginx

# 6. Установка и базовая защита через CrowdSec
sudo apt install crowdsec -y
curl -s [https://install.crowdsec.net](https://install.crowdsec.net) | sudo bash
sudo apt install crowdsec-firewall-bouncer-iptables -y
sudo cscli collections install crowdsecurity/nginx
sudo systemctl restart crowdsec

# 7. Настройка Фаервола (UFW)
sudo apt install ufw
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 22/tcp  # Оставьте ваш реальный порт SSH, если еще не изолировали его
sudo ufw enable

# 8. Установка и запуск wstunnel
sudo wget https://github.com/erebe/wstunnel/releases/download/v10.6.1/wstunnel_10.6.1_linux_amd64.tar.gz
sudo tar -xzvf wstunnel_10.6.1_linux_amd64.tar.gz
sudo mv wstunnel /usr/local/bin/
sudo chmod +x /usr/local/bin/wstunnel
```
upd:
Закрываем сервак для всех кроме клоудфаер
В чём прикол: 
Из-за утечек домена и IP (например, через DNS или старые логи), боты и цензоры могут постучаться напрямую на ваш реальный IP-адрес, подсунув в SNI ваш валидный домен. 
Если порты открыты на весь мир, Nginx честно схавает этот запрос и ответит. 
Это огромная дыра в безопасности, которая полностью деанонимизирует сервер для ТСПУ и сканеров (Censys, Shodan).  Раз наш VLESS-трафик и SSH (wstunnel) и так полностью завернуты в CDN, мы можем превратить сервер в абсолютную черную дыру для всех левых прохожих. Мы оставим в живых на фаерволе ТОЛЬКО подсети самой Cloudflare. 
Теперь любые прямые попытки ботов постучаться по IP (даже с правильным SNI) наткнутся на глухой таймаут (DROP) на уровне ядра Linux, потому что до Nginx пакет просто не долетит. 
Сервер полностью пропадает с радаров глобального интернета.
Как сделать:
Сносим старые глобальные правила в UFW, чтобы порты 80 и 443 больше не светились на весь мир: 
```bash
sudo ufw delete allow 80/tcp
sudo ufw delete allow 443/tcp
```
Пишем «умный» автоматический скрипт с комментариями правил, чтобы ufw не забивался дубликатами. 
Создаем файл:

sudo nano /etc/cron.weekly/cloudflare-ufw

Вставляем Bash-код:
```bash
#!/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

CF_IPV4=$(curl -s https://www.cloudflare.com/ips-v4)
CF_IPV6=$(curl -s https://www.cloudflare.com/ips-v6)

if [ -z "$CF_IPV4" ]; then
    echo "Ошибка: не удалось загрузить IP Cloudflare"
    exit 1
fi


ufw status numbered | grep 'CF_AUTO' | awk -F"[][]" '{print $2}' | sort -rn | xargs -I {} ufw --force delete {}


for ip in $CF_IPV4; do
    ufw allow from "$ip" to any port 80 proto tcp comment 'CF_AUTO'
    ufw allow from "$ip" to any port 443 proto tcp comment 'CF_AUTO'
done


for ip in $CF_IPV6; do
    ufw allow from "$ip" to any port 80 proto tcp comment 'CF_AUTO'
    ufw allow from "$ip" to any port 443 proto tcp comment 'CF_AUTO'
done

ufw reload
```
Делаем файл исполняемым и запускаем первый раз ручками:
```bash
sudo chmod +x /etc/cron.weekly/cloudflare-ufw
sudo /etc/cron.weekly/cloudflare-ufw
```
Скрипт сам встал в еженедельный крон и будет обновлять айпишники Cloudflare в фоне.
