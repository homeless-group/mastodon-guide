## Mastodon Guide

### Prepare

- Git
- Nginx
- Docker
- Docker Compose

```bash
# Currently I used Oracle Free Cloud Instance to run this application.
# VM Configuration
uname -a
Linux 4.15.0-1029-oracle #32-Ubuntu SMP Fri Nov 8 06:40:49 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux

# memory usage
free -h
free -h
              total        used        free      shared  buff/cache   available
Mem:           982M        680M         88M        6.8M        213M        156M
Swap:          2.0G        543M        1.5G

# cpu usage
top -c
top - 15:00:41 up 4 days, 21:35,  9 users,  load average: 1.02, 1.26, 1.39
Tasks: 147 total,   2 running, 102 sleeping,   0 stopped,   0 zombie
%Cpu0  :  2.7 us,  5.8 sy,  0.0 ni, 62.9 id,  0.0 wa,  0.0 hi,  0.0 si, 28.6 st
%Cpu1  :  4.2 us,  6.0 sy,  0.0 ni, 60.2 id,  0.0 wa,  0.0 hi,  0.0 si, 29.6 st
KiB Mem :  1005740 total,    93400 free,   699632 used,   212708 buff/cache
KiB Swap:  2097148 total,  1540604 free,   556544 used.   157672 avail Mem

# docker run stats
docker stats
CONTAINER ID        NAME                   CPU %               MEM USAGE / LIMIT     MEM %               NET I/O             BLOCK I/O           PIDS
08112c35856d        mastodon_sidekiq_1     10.50%              98.77MiB / 982.2MiB   10.06%              30.9MB / 48MB       137MB / 0B          14
7914ade54e65        mastodon_streaming_1   0.02%               15.12MiB / 982.2MiB   1.54%               200kB / 169kB       138MB / 0B          19
cfb272cb4644        mastodon_web_1         280.17%             290.7MiB / 982.2MiB   29.60%              16.6MB / 8.25MB     290MB / 201kB       31
ddac7786dd51        mastodon_redis_1       0.28%               1.859MiB / 982.2MiB   0.19%               48.2MB / 30MB       16.9MB / 2.67MB     4
a224f192f8d9        mastodon_db_1          0.00%               12.44MiB / 982.2MiB   1.27%               3.12MB / 16.1MB     64.7MB / 38.1MB     11

```

### Installation

- Clone Repository

```bash
mkdir ~/apps
cd ~/apps
git clone https://github.com/tootsuite/mastodon
cd mastodon
```

- Edit `docker-compose.yml`

```bash
# vim docker-compose.yml
# skipping local compile, so please making yml build . line is commented. Totally comment 3 places
# If you want choose specific mastodon version, please add version below of related images, such as image: tootsuite/mastodon:3.0.1

db:
    # [...]
    volumes:
      - ./postgres:/var/lib/postgresql/data
redis:
    # [...]
    volumes:
      - ./redis:/data      
web:
#    build: .
    image: tootsuite/mastodon
    # [...]
    volumes:
      - ./public/system:/mastodon/public/system
      - ./public/assets:/mastodon/public/assets
      - ./public/packs:/mastodon/public/packs
  streaming:
#    build: .
    image: tootsuite/mastodon
    # [...]
  sidekiq:
#    build: .
    image: tootsuite/mastodon
    volumes:
      - ./public/system:/mastodon/public/system
      - ./public/packs:/mastodon/public/packs
```

- Prepare env configuration

```bash
cp .env.production.sample .env.production

# vim .env.production
## generate screct
# 1. SECRET_KEY_BASE:
sed -i "s/SECRET_KEY_BASE=$/&$(docker-compose run --rm web bundle exec rake secret)/" .env.production

# 2. OTP_SECRET:
sed -i "s/OTP_SECRET=$/&$(docker-compose run --rm web bundle exec rake secret)/" .env.production

# 3. vapid keys
docker-compose run --rm web bundle exec rake mastodon:webpush:generate_vapid_key

```

- Complete `.env.production`

```bash
LOCAL_DOMAIN=example.com
SINGLE_USER_MODE=false
SECRET_KEY_BASE=
OTP_SECRET=
VAPID_PRIVATE_KEY=
VAPID_PUBLIC_KEY=

# Database settings
DB_HOST=db
DB_PORT=5432
DB_NAME=postgres
DB_USER=postgres
DB_PASS=

# Redis settings
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=

# Mail settings
# current use Ali Mail with TLS
# And now have the issue of email was recognized a spam
SMTP_SERVER=smtp.mxhichina.com
SMTP_PORT=465
SMTP_LOGIN=notifications@example.com
SMTP_PASSWORD=your_smtp_password
SMTP_AUTH_METHOD=plain
SMTP_OPENSSL_VERIFY_MODE=none
SMTP_FROM_ADDRESS=Mastodon <notifications@example.com>
SMTP_TLS=true
```

- Nginx Proxy

```bash
# current I use local nginx proxy, not docker mode.
# It just my habit easy to control

# almost same with source nginx sample
# sudo vim /etc/nginx/site-available/mastodon

map $http_upgrade $connection_upgrade {
  default upgrade;
  ''      close;
}

server {
  listen 80;
  listen [::]:80;
  server_name example.com;
  # Useful for Let's Encrypt
  location /.well-known/acme-challenge/ { root /usr/share/nginx/html; allow all; }
  location / { return 301 https://$host$request_uri; }
}

server {
  listen 443 ssl http2;
  listen [::]:443 ssl http2;
  server_name example.com;

  ssl_protocols TLSv1.2;
  ssl_ciphers HIGH:!MEDIUM:!LOW:!aNULL:!NULL:!SHA;
  ssl_prefer_server_ciphers on;
  ssl_session_cache shared:SSL:10m;

  ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

  keepalive_timeout    70;
  sendfile             on;
  client_max_body_size 0;

  root /home/mastodon/live/public;

  gzip on;
  gzip_disable "msie6";
  gzip_vary on;
  gzip_proxied any;
  gzip_comp_level 6;
  gzip_buffers 16 8k;
  gzip_http_version 1.1;
  gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

  add_header Content-Security-Policy "default-src 'none'; font-src 'self'; media-src 'self'; style-src 'self' 'unsafe-inline'; script-src 'self'; img-src 'self' data:; connect-src 'self' wss://mastodon.partecipa.digital; frame-ancestors 'none';";
  add_header Referrer-Policy "no-referrer";
  add_header Strict-Transport-Security "max-age=31536000";

  location / {
    try_files $uri @proxy;
  }

  location ~ ^/(packs|system/media_attachments/files|system/accounts/avatars) {
    add_header Cache-Control "public, max-age=31536000, immutable";
    try_files $uri @proxy;
  }

  location @proxy {
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header Proxy "";
    proxy_pass_header Server;

    proxy_pass http://web:3000;
    proxy_buffering off;
    proxy_redirect off;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $connection_upgrade;

    tcp_nodelay on;
  }

  location /api/v1/streaming {
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header Proxy "";

    proxy_pass http://streaming:4000;
    proxy_buffering off;
    proxy_redirect off;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $connection_upgrade;

    tcp_nodelay on;
  }

  error_page 500 501 502 503 504 /500.html;
}


sudo ln -s /etc/nginx/site-available/mastodon /etc/nginx/site-enabled/mastodon
```

- Mastodon Setup

```bash
# chown public folder permission
sudo chown -r 991:991 public/

# initialize database
docker-compose run --rm web bundle exec rake db:migrate

# pre-compile web assets
docker-compose run --rm web bundle exec rake assets:precompile

# start with daemon
docker-compose up -d
```

- Mastodon Usage

```bash
# If you want to create a user without web instead of terminal
docker-compose run --rm web tootctl accounts create your-username --email your-email-address --confirmed --role admin

# create normal user
docker-compose run --rm web bundle exec rake mastodon:confirm_email USER_EMAIL=your_email_address

# grant a role
docker-compose run --rm web bundle exec rake mastodon:make_admin USERNAME=your_mastodon_instance_user
```

### Tips

_P.S. This SMTP setting hang with me almost one day, because the parameter of SMTP_DELIVERY_METHOD value SHOULD NOT config, but docker container still load the old environment variables, And I have not found an effective way to solve._

### References

[How to Install Mastodon on Ubuntu 16.04](https://www.linode.com/docs/applications/messaging/install-mastodon-on-ubuntu-1604/)