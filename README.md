# Home_work_prometheus
#### 1. На виртуальной машине 1-HV, с помощью docker-compose был развернут:
   - Nginx
   - MySql
   - WordPress
   - Node-exporter
   - cadvisor
```
services:
  db:
    image: mysql:8.0
    volumes:
      - db_data:/var/lib/mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: somewordpress
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: wordpress
    command: '--authentication_policy=mysql_native_password'
    networks:
      - wp

  wordpress:
    depends_on:
      - db
    image: wordpress:5.1.1-fpm-alpine
    volumes:
      - wordpress_data:/var/www/html
    restart: always
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: wordpress
      WORDPRESS_DB_NAME: wordpress
    networks:
      - wp

  webserver:
    depends_on:
      - wordpress
    image: nginx:1.15.12-alpine
    container_name: webserver
    restart: unless-stopped
    ports:
      - "8000:80"
    volumes:
      - wordpress_data:/var/www/html
      - .nginx.conf:/etc/nginx/conf.d
      - .nginx.conf/.htpasswd:/etc/nginx/.htpasswd
    networks:
      - wp

  node-exporter:
    image: prom/node-exporter
    restart: on-failure
    container_name: node-fsk
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
      - /etc/localtime:/etc/localtime
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - --collector.filesystem.mount-points-exclude
      - "^/(sys|proc|dev|host|etc|rootfs/var/lib/docker/containers|rootfs/var/lib/docker/overlay2|rootfs/run/docker/netns|rootfs/var/lib/docker/aufs)($$|/)"
    expose:
      - "9100"
    networks:
      - wp

  cadvisor-exporter:
        container_name: cadvisor-exporter
        image: google/cadvisor
        expose:
          - "8080"
        volumes:
          - "/:/rootfs:ro"
          - "/var/run:/var/run:rw"
          - "/sys:/sys:ro"
          - "/var/lib/docker/:/var/lib/docker:ro"
        restart: unless-stopped
        networks:
           - wp

volumes:
  db_data: {}
  wordpress_data: {}

networks:
  wp:
```
#### 2. На виртуальной машине 2-HV, с помощью docker-compose был развернут:
   - Prometheus
   - Blackbox-exporter
   - Alertmanager
   - Telegram-bot
```
services:

  prometheus:
    image: prom/prometheus
    container_name: prometheus
    restart: on-failure
    volumes:
      - ./conf/:/etc/prometheus/
      - prometheus_data:/prometheus
      - /etc/localtime:/etc/localtime
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/usr/share/prometheus/console_libraries'
      - '--web.console.templates=/usr/share/prometheus/consoles'
      - '--web.enable-lifecycle'
    ports:
      - 9090:9090
    networks:
      - prom
  blackbox-exporter:
    image: prom/blackbox-exporter:latest
    container_name: blackbox-exporter
    volumes:
      - ./blackbox-exporter/:/config/
    command:
      - '--config.file=/config/blackbox.yml'
    restart: unless-stopped
    networks:
      - prom
    mem_limit: 128m
    mem_reservation: 64m
    logging:
        driver: "json-file"
        options:
          max-size: "5m"

  telegram-bot:
    image: metalmatze/alertmanager-bot:0.4.3
    container_name: telegram-bot
    restart: on-failure
    ports:
      - 8080:8080
    volumes:
      - bot_db:/data/
    command:
    - --alertmanager.url=http://alertmanager:9093
    - --log.level=info
    - --store=bolt
    - --bolt.path=/data/bot.db
    environment:
      TELEGRAM_ADMIN: ""
      TELEGRAM_TOKEN: 
    networks:
      - prom

  alertmanager:
    image: prom/alertmanager
    container_name: alertmanager
    ports:
      - 9093:9093
    volumes:
      - ./alertmanager/:/etc/alertmanager/
    restart: always
    networks:
      - prom
    command:
      - '--config.file=/etc/alertmanager/config.yml'
      - '--storage.path=/alertmanager'

networks:
    prom:
volumes:
    prometheus_data:
    bot_db:
```
#### 3. Кофигурация Alertmanager
**Prometheus и Alertmanager** настроен так, что при отстановке одного из контейнеров стека CMS (mysql,nginx,wordpress), будет отправленно оповещение в канал telegram. Так же настроен **blackbox-exporter** на проверку доступности сайта, при не доступности будет отправлено оповещение в канал telegram. Конфигурационные файлы [Prometheus](/GAP-1/Prometheus), [Alertmanager](/GAP-1/Alertmanager), [Blackbox-exporter](/GAP-1/blackbox-exporter).
#### 4. Задание со *
Для доступа к метрикам **node-exporter и cadvisor по одному** порту **8000** на вирутальной машине 1-HV, был переконфигурирован **nginx** и добавлены два блока 
**location /node-metrics** и **location /docker-metrics**. А так же была реализованна простая авторизация по имени поьзователя и паролю *HTTP Basic Authentication*:
```
server {
        listen 80;
        listen [::]:80;

        server_name 10.43.2.66;

        index index.php index.html index.htm;

        root /var/www/html;

        location ~ /.well-known/acme-challenge {
                allow all;
                root /var/www/html;
        }

        location / {
                try_files $uri $uri/ /index.php$is_args$args;
        }

        location ~ \.php$ {
                try_files $uri =404;
                fastcgi_split_path_info ^(.+\.php)(/.+)$;
                fastcgi_pass wordpress:9000;
                fastcgi_index index.php;
                include fastcgi_params;
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                fastcgi_param PATH_INFO $fastcgi_path_info;
        }

        location /node-metrics {
                proxy_pass http://10.4.2.66:9100/metrics;
                auth_basic "nginx";
                auth_basic_user_file "/etc/nginx/.htpasswd";
        }


        location /docker-metrics {
                proxy_pass http://10.4.2.66:8080/metrics;
                auth_basic "nginx";
                auth_basic_user_file "/etc/nginx/.htpasswd";
        }
        
        location ~ /\.ht {
                deny all;
        }

        location = /favicon.ico {
                log_not_found off; access_log off;
        }
        location = /robots.txt {
                log_not_found off; access_log off; allow all;
        }
        location ~* \.(css|gif|ico|jpeg|jpg|js|png)$ {
                expires max;
                log_not_found off;
        }
}

```
# Home_work_grafana
#### 1. На виртуальной машине 2-HV, с помощью модифицированного docker-compose был развернут и добавлен в стек контейнер "Grafana":
   - Prometheus
   - Blackbox-exporter
   - Alertmanager
   - Telegram-bot
   - **Garafana**
```
...
service:
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    volumes:
      - grafana_data:/var/lib/grafana
    ports:
      - "3000:3000"
    networks:
      - prom
    restart: always
```
#### 2. Исходя из задания были созданны две папки infra и app, с вложенными dashbord'ами [infra](/GAP-2/dashboard-infra/infra.jpg) и [app](/GAP-2/dashboard-app/app.jpg)
#### 3. Задание со * 1
С помощью dashbord'а **app** и панели **Container running** 
 - Query:
```
count(rate(container_last_seen{job="node-fsk-docker-container",container_label_com_docker_compose_project="wordpress"}[1m])) by (instance)
```
был настроен **Alert**, если умрет один из контейнеров стека CMS то будет отправлено 
![alt text](https://github.com/ckyhu/Home_work_prometheus/blob/main/GAP-2/Alert/alert.jpg)

оповещение в **Telegram**
![alt text](https://github.com/ckyhu/Home_work_prometheus/blob/main/GAP-2/Alert/telegram.jpg)

#### 4. Задание со * 2
На dashbord'е [DrillDown](/GAP-2/dashboard-drilldown/drilldown.jpg) избражена сводная информация по docker хосту и стеку CMS. При нажатии на панели **Load CPU in Host** и **RAM used** происходит переход в dashbord **infra** с более детальной информацией, для работы данной функции была использована **Data Links** в свойствах панели.
При наведении курсора на панели **Container running in host** и **Container stack CMS memory usege** в верхнем углу есть ссылка на dashbord **infra**.
![alt text](https://github.com/ckyhu/Home_work_prometheus/blob/main/GAP-2/dashboard-drilldown/link.jpg)

