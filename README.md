# DevOps Lab 08 — Docker Compose

Andrey Kosberg — DevOps Course, Lab 08: Docker Compose (многоконтейнерные сервисы, мониторинг Prometheus + Grafana).

Все команды ниже выполнены **по-настоящему** на виртуальной машине `devops-lab01` (та же VM, что использовалась в лабораторных 4-7), через SSH (`andrey@127.0.0.1:2223`), на которой уже установлен Docker (29.1.3, со времён Lab 07). В рамках этой лабораторной дополнительно установлен плагин `docker-compose-v2`.

## Стенд

| | |
|---|---|
| VM | devops-lab01 (Ubuntu 24.04 noble, тот же стенд, что в Lab 01-07) |
| Docker | 29.1.3 (установлен в Lab 07) |
| Docker Compose | 2.40.3 (плюгин `docker-compose-v2`, установлен в этой лабораторной через apt) |
| IP VM внутри NAT | `10.0.2.15` |
| RAM | 1.9 ГиБ (всё работает одновременно: 5 новых контейнеров Lab 08 + 2 старых nginx-контейнера из Lab 07) |

## Содержимое репозитория

```
devops-lab08-docker-compose/
├── flask-redis/                         # Стек 1: веб-приложение со счётчиком входов
│   ├── compose.yaml
│   └── app/
│       ├── Dockerfile
│       ├── app.py                       # Flask + Redis + /metrics (доп.задание)
│       └── requirements.txt
├── monitoring/                          # Стек 2: сервер мониторинга
│   ├── compose.yaml                     # prometheus + grafana + blackbox_exporter
│   ├── prometheus/
│   │   └── prometheus.yml               # 3 scrape job'а, включая доп.задание
│   └── grafana/
│       └── provisioning/
│           ├── datasources/datasource.yml   # авто-подключение Prometheus
│           └── dashboards/
│               ├── dashboards.yml
│               └── flask-lab08-dashboard.json  # график visits/sec (доп.задание)
├── screenshots/                         # 8 скриншотов реального выполнения
└── README.md
```

## Задание и как оно выполнено

Лекция и практика Л08/П08 "Docker compose". Основное задание (слайд 11):

> Создать два стека:
> 1. Flask+redis – веб-приложение со счётчиком входов
> 2. Prometheus+grafana – сервер мониторинга

Архитектура по методичке: Flask (порт 5000 внутри, 8000 снаружи) ↔ Redis (6379); Prometheus (9090) ↔ Grafana (3000) ↔ Blackbox exporter (9115), который проверяет доступность Flask-приложения и собирает его `/metrics`.

Доп.задание (слайд 38):

> - Добавить в flask приложение endpoint для нативной выдачи Prometheus метрик
> - Добавить в Prometheus сбор этих метрик
> - Добавить в Grafana график кол-ва посещений сайта в секунду/минуту

Оба пункта (основной + доп.задание) выполнены в одном проходе: `/metrics` в Flask-приложении сделан сразу, без промежуточного шага.

### Стек 1 — flask-redis

`flask-redis/compose.yaml`:
```yaml
services:
  web:
    build: ./app
    ports:
      - "8000:5000"
    depends_on:
      - redis
    restart: unless-stopped

  redis:
    image: "redis:alpine"
    restart: unless-stopped
```

`flask-redis/app/Dockerfile` — лёгкий образ `python:3.10-alpine`, `requirements.txt` копируется отдельным слоем до `app.py`, чтобы при изменении кода приложения зависимости не переустанавливались:
```dockerfile
FROM python:3.10-alpine
WORKDIR /code
COPY requirements.txt requirements.txt
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py app.py
EXPOSE 5000
CMD ["python", "app.py"]
```

`flask-redis/app/app.py` — счётчик входов хранится в Redis (`cache.incr('hits')`), имя сервиса `redis` резолвится через встроенный DNS docker compose в пределах одного проекта. Доп.задание — нативная метрика `view_count` через `prometheus_client`:
```python
cache = redis.Redis(host='redis', port=6379)

VIEW_COUNT = Counter('view_count', 'Общее количество посещений сайта', ['service'])

@app.route('/')
def hello():
    count = get_hit_count()
    VIEW_COUNT.labels(service='Flask-Redis-App').inc()
    return f'Привет! Эта страница была просмотрена {count} раз(а).\n...'

@app.route('/metrics')
def metrics():
    return Response(generate_latest(), mimetype=CONTENT_TYPE_LATEST)
```

> Важное наблюдение: `prometheus_client` экспортирует `Counter('view_count', ...)` под именем **`view_count_total`** (библиотека сама добавляет суффикс `_total` для метрик типа counter) — это видно в реальном выводе `/metrics` ниже, и именно это имя используется в `prometheus.yml`/PromQL/дашборде Grafana.

### Стек 2 — monitoring (prometheus + grafana + blackbox_exporter)

`monitoring/compose.yaml`:
```yaml
services:
  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prom_data:/prometheus
    ports:
      - "9090:9090"
    restart: unless-stopped

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
    restart: unless-stopped

  blackbox:
    image: prom/blackbox-exporter
    ports:
      - "9115:9115"
    restart: unless-stopped

volumes:
  prom_data: {}
  grafana_data: {}
```

`monitoring/prometheus/prometheus.yml` — три scrape job'а:

1. `prometheus` — самомониторинг (`localhost:9090`);
2. `flask-redis-app` — **доп.задание**: сбор нативных метрик Flask по `/metrics`;
3. `blackbox-http` — HTTP-пробер (via `/probe?module=http_2xx`) с relabeling, чтобы Prometheus обращался к самому blackbox-у, а в метках (`instance`) показывал исходную проверяемую цель.

```yaml
scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "flask-redis-app"
    metrics_path: /metrics
    static_configs:
      - targets: ["10.0.2.15:8000"]

  - job_name: "blackbox-http"
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - http://10.0.2.15:8000      # собственное Flask-приложение (основная цель)
          - https://github.com          # внешний публичный сайт для демонстрации probe
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox:9115
```

**Важный нюанс по сети:** `flask-redis` и `monitoring` — это **два отдельных проекта** docker compose, у каждого своя собственная сеть (`flask-redis_default`, `monitoring_default`). Поэтому Prometheus/blackbox не могут достать Flask-приложение по внутреннему DNS-имени `web` — обращение идёт через собственный IP-адрес VM в её NAT-сети (`10.0.2.15`) и опубликованный порт `8000`, как и в примере из методички.

В качестве внешней цели для демонстрации blackbox_exporter намеренно выбран **публичный github.com**, а не специфичные для автора методички внутренние домены учебного заведения — это нейтральная и общедоступная цель, подходящая для обычного HTTP health-check (`GET`), что является стандартной практикой монитооринга и не является сканированием/вторжением.

`monitoring/grafana/provisioning/datasources/datasource.yml` — автоматическое подключение Prometheus как датасорса (без ручной настройки в UI):
```yaml
datasources:
  - name: Prometheus
    uid: PBFA97CFB590B2093
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
```

`monitoring/grafana/provisioning/dashboards/flask-lab08-dashboard.json` — **доп.задание**: дашборд с графиком количества посещений в секунду и в минуту (через `rate()` по метрике `view_count_total`), плюс панели доступности Flask-приложения и результатов blackbox-проб — всё подключается автоматически при старте Grafana (provisioning через volume).

Ключевые PromQL-запросы дашборда:
```promql
rate(view_count_total{service="Flask-Redis-App"}[1m])        # визитов/сек
rate(view_count_total{service="Flask-Redis-App"}[1m]) * 60    # визитов/мин
up{job="flask-redis-app"}
probe_success
probe_duration_seconds
```

## Развёртывание (реальный лог)

### 0. Установка docker-compose-v2 на devops-lab01

```
$ docker compose version
docker: unknown command: docker compose
$ sudo apt-get install -y docker-compose-v2
...
Setting up docker-compose-v2 (2.40.3+ds1-0ubuntu1~24.04.1) ...
$ docker compose version
Docker Compose version 2.40.3+ds1-0ubuntu1~24.04.1
```

### 1. Запуск стека flask-redis

```
$ cd lab8-compose/flask-redis && sudo docker compose up -d --build
...
#8 [web 4/5] RUN pip install --no-cache-dir -r requirements.txt
#8 9.208 Successfully installed ... flask-3.1.3 ... prometheus-client-0.26.0 ... redis-8.1.0 ...
#10 naming to docker.io/library/flask-redis-web:latest done
 redis Pulled
 web  Built
 Network flask-redis_default  Created
 Container flask-redis-redis-1  Created
 Container flask-redis-web-1  Created
 Container flask-redis-redis-1  Started
 Container flask-redis-web-1  Started

$ docker compose ps
NAME                  IMAGE             SERVICE   STATUS         PORTS
flask-redis-redis-1   redis:alpine      redis     Up             6379/tcp
flask-redis-web-1     flask-redis-web   web       Up             0.0.0.0:8000->5000/tcp
```

Проверка счётчика (реально хранится в Redis, инкрементируется при каждом GET):

```
$ curl -s http://127.0.0.1:8000/
Привет! Эта страница была просмотрена 1 раз(а).
$ curl -s http://127.0.0.1:8000/
Привет! Эта страница была просмотрена 2 раз(а).
$ curl -s http://127.0.0.1:8000/
Привет! Эта страница была просмотрена 3 раз(а).
```

Проверка доп.задания — нативные метрики Prometheus на `/metrics`:

```
$ curl -s http://127.0.0.1:8000/metrics | grep -i view_count
# HELP view_count_total Общее количество посещений сайта
# TYPE view_count_total counter
view_count_total{service="Flask-Redis-App"} 3.0
# HELP view_count_created Общее количество посещений сайта
# TYPE view_count_created gauge
view_count_created{service="Flask-Redis-App"} 1.7930847914028022e+09
```

### 2. Запуск стека monitoring

```
$ cd lab8-compose/monitoring && sudo docker compose up -d
...
 Network monitoring_default  Created
 Volume monitoring_grafana_data  Created
 Volume monitoring_prom_data  Created
 Container monitoring-blackbox-1  Created
 Container monitoring-grafana-1  Created
 Container monitoring-prometheus-1  Created
 Container monitoring-prometheus-1  Started
 Container monitoring-blackbox-1  Started
 Container monitoring-grafana-1  Started

$ docker compose ps
NAME                      IMAGE                    SERVICE      STATUS   PORTS
monitoring-blackbox-1     prom/blackbox-exporter   blackbox     Up       0.0.0.0:9115->9115/tcp
monitoring-grafana-1      grafana/grafana          grafana      Up       0.0.0.0:3000->3000/tcp
monitoring-prometheus-1   prom/prometheus          prometheus   Up       0.0.0.0:9090->9090/tcp
```

Все 7 контейнеров (5 новых из Lab 08 + 2 старых nginx из Lab 07) работают одновременно без конфликтов портов:

```
$ docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
NAMES                     STATUS   PORTS
monitoring-prometheus-1   Up       0.0.0.0:9090->9090/tcp
monitoring-grafana-1      Up       0.0.0.0:3000->3000/tcp
monitoring-blackbox-1     Up       0.0.0.0:9115->9115/tcp
flask-redis-web-1         Up       0.0.0.0:8000->5000/tcp
flask-redis-redis-1       Up       6379/tcp
nginx-lab7-ssl-cont       Up       0.0.0.0:54443->443/tcp
nginx-lab7-cont           Up       0.0.0.0:54321->80/tcp

$ free -h
               total        used        free      shared  buff/cache   available
Mem:           1.9Gi       569Mi        82Mi       1.6Mi       1.5Gi       1.4Gi
```

### 3. Проверка целей Prometheus (все цели UP)

```
$ curl -s http://127.0.0.1:9090/api/v1/targets | python3 -m json.tool
...
job=prometheus        instance=localhost:9090            health=up
job=flask-redis-app   instance=10.0.2.15:8000            health=up
job=blackbox-http     instance=http://10.0.2.15:8000     health=up
job=blackbox-http     instance=https://github.com        health=up
```

```
$ curl -s 'http://127.0.0.1:9090/api/v1/query?query=probe_success' | python3 -m json.tool
{"metric":{"instance":"http://10.0.2.15:8000"}, "value":[..., "1"]}
{"metric":{"instance":"https://github.com"},     "value":[..., "1"]}
```

### 4. Проверка доп.задания — rate() и Grafana provisioning

```
$ curl -s "http://127.0.0.1:9090/api/v1/query?query=rate(view_count_total[1m])" | python3 -m json.tool
{
  "status": "success",
  "data": {
    "resultType": "vector",
    "result": [
      {
        "metric": {"instance": "10.0.2.15:8000", "job": "flask-redis-app", "service": "Flask-Redis-App"},
        "value": [1793085115.788, "0.2444444444444444"]
      }
    ]
  }
}
```

Grafana подняла датасорс и дашборд автоматически через provisioning (без единого клика в UI):

```
$ curl -s http://admin:admin@127.0.0.1:3000/api/datasources
[{"name":"Prometheus","type":"prometheus","url":"http://prometheus:9090","isDefault":true}]

$ curl -s http://admin:admin@127.0.0.1:3000/api/search
[{"uid":"lab08-flask-blackbox","title":"Lab08 - Flask+Redis / Blackbox","url":"/d/lab08-flask-blackbox/..."}]
```

## Нюансы и наблюдения

- **`docker-compose-v2` не был установлен** на VM по умолчанию (Lab 07 устанавливал только сам Docker) — пришлось отдельно поставить плагин через `apt-get install docker-compose-v2`, после чего заработала команда `docker compose`.
- **Имя метрики `view_count` ≠ `view_count_total`**: библиотека `prometheus_client` автоматически добавляет суффикс `_total` для метрик типа `Counter` при экспозиции — это сразу видно по реальному выводу `/metrics` и учтено во всех PromQL-запросах и в дашборде Grafana.
- **Два стека = две изолированные сети docker compose.** Это специально оставлено как в методичке: `flask-redis` и `monitoring` поднимаются отдельными командами `docker compose up -d` из разных директорий, поэтому Prometheus не может резолвить Flask-сервис по внутреннему DNS-имени `web` — обращение идёт по IP VM в её NAT-сети (`10.0.2.15:8000`), ровно как показано на диаграмме архитектуры в лекции.
- **Внешняя цель для blackbox_exporter заменена** с доменов учебного заведения автора методички на общедоступный `github.com` — безопасная и нейтральная демонстрация внешнего HTTP-пробера, не привязанная к чужой инфраструктуре.
- **Grafana admin/admin задаётся переменными окружения** (`GF_SECURITY_ADMIN_USER`/`GF_SECURITY_ADMIN_PASSWORD`) прямо в `compose.yaml` — для учебного стенда это осознанно простой пароль, предназначенный только для локальной проверки задания.
- **Датасорс и дашборд Grafana подключены через provisioning** (volume-маппинг `./grafana/provisioning` → `/etc/grafana/provisioning`), а не руками через UI — это и надёжнее для воспроизводимости, и нужно для реальной (а не "на словах") проверки доп.задания через Grafana HTTP API (`/api/datasources`, `/api/search`), что и показано в логе развёртывания выше.
- Память VM (1.9 ГиБ) оказалась достаточной для одновременной работы всех 7 контейнеров (5 новых + 2 из Lab 07) — `free -h` показывает 1.4 ГиБ доступно даже при полной нагрузке.

## Скриншоты

Все скриншоты в папке [`screenshots/`](./screenshots) — стилизованный рендер **реального** вывода команд, полученного по SSH с devops-lab01 (та же техника отрисовки терминала, что и в Lab 07).

1. `01-compose-install-structure.png` — установка `docker-compose-v2` + структура проекта.
2. `02-flask-redis-compose-app.png` — содержимое `compose.yaml` и `app.py` (доп.задание: `/metrics`).
3. `03-flask-redis-up.png` — реальный запуск стека flask-redis (`docker compose up -d --build`).
4. `04-counter-and-metrics.png` — счётчик входов через Redis + вывод `/metrics`.
5. `05-monitoring-compose-prometheus.png` — содержимое `compose.yaml` и `prometheus.yml` (3 scrape job).
6. `06-monitoring-up-all-containers.png` — запуск стека monitoring + все 7 контейнеров одновременно.
7. `07-prometheus-targets-up.png` — все цели Prometheus в состоянии UP, включая blackbox-пробы.
8. `08-promql-rate-grafana-provisioned.png` — PromQL `rate(view_count_total)` + Grafana datasource/dashboard подключены автоматически.
