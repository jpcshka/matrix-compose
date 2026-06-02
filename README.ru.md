# Matrix сервер на базе Synapse

Готовая конфигурация для поднятия Matrix homeserver через Docker Compose.

## Оглавление

1. [Стек](#стек)
2. [Структура репозитория](#структура-репозитория)
3. [Требования](#требования)
4. [Подготовка DNS](#подготовка-dns)
5. [Быстрый старт](#быстрый-старт)
6. [Описание компонентов](#описание-компонентов)
    - [Synapse](#synapse)
    - [Caddy](#caddy)
    - [LiveKit](#livekit)
    - [lk-jwt-service](#lk-jwt-service)


---

## Стек 

- **[Matrix Synapse](https://github.com/element-hq/synapse)**: Основной homeserver, разрабатывается The Matrix.org Foundation на языке Python.
- **[Caddy](https://github.com/caddyserver/caddy)**: Обратный прокси-сервер, выпускает и управляет сертификатами Let's Encrypt. Разрабатывается Мэттом Холтом на Go.
- **[LiveKit](https://github.com/livekit/livekit)**: Медиа сервер для MatrixRTC, написан на Go компанией LiveKit Inc.
- **[lk-jwt-service](https://github.com/element-hq/lk-jwt-service)**: Сервис авторизации для MatrixRTC, разрабатывается The Matrix.org Foundation на языке Go.


---


## Структура репозитория

```text
.
├── caddy/              # Конфигурация Caddy (Caddyfile)
├── livekit/            # Пример конфигурации livekit (livekit.yaml.example)
├── synapse/            # Synapse с S3 поддержкой (Dockerfile)
├── static/             # Статические файлы для заглушки (html, css, js)
├── .env.example        # Пример переменных окружения
└── compose.yaml        # Основной файл запуска
```


---


## Требования

- Linux сервер (рекомендуется Debian 13)
- Доменное имя
- Открытые порты:
  - `80/tcp`
  - `443/tcp`
  - `443/udp`
  - `7881/tcp` (опционально)
  - `50100-50101/udp` (опционально)

> Список используемых портов зависит от конфигурации MatrixRTC (LiveKit).

### Рекомендуемая конфигурация сервера

| Ресурс | Рекомендация |
|---|---|
| CPU | 2-4 vCPU |
| RAM | 2-8 GB |
| Диск | 20+ GB |
| Сеть | Публичный IPv4 |

> Для MatrixRTC рекомендуется минимум 4 GB RAM.


---

## Подготовка DNS

Сделайте А-записи для доменов Matrix homeserver и MatrixRTC. Укажите свой IP-адрес.

```dns
example.com          A    203.0.113.1 
matrix.example.com   A    203.0.113.1
livekit.example.com  A    203.0.113.1
```

> Все сервисы размещаются на одном сервере, поэтому DNS-записи должны указывать на один IP-адрес.

---


## Быстрый старт

1. Скопируйте и заполните переменные окружения:

```bash
cp .env.example .env
```

2. Прочитайте раздел [Описание компонентов](#описание-компонентов) и настройте каждый сервис согласно документации.

3. Запустите:

```bash
docker compose up -d
```

> Перед запуском убедитесь, что DNS-записи настроены и порты открыты — Caddy не выпустит сертификаты без доступа извне.

---


## Описание компонентов

### Synapse

Matrix homeserver на базе официального образа [matrixdotorg/synapse](https://hub.docker.com/r/matrixdotorg/synapse) с добавленным модулем [synapse-s3-storage-provider](https://github.com/matrix-org/synapse-s3-storage-provider) для хранения медиа в S3-совместимых хранилищах.

Конфиги, ключи и медиа хранятся в `/var/lib/matrix-compose/data`.

Создайте директорию:
```bash
mkdir -p /var/lib/matrix-compose/data
```

Сгенерируйте файлы конфигураций:

```bash
docker run -it --rm \
    --mount type=bind,src=/var/lib/matrix-compose/data,dst=/data \
    -e SYNAPSE_SERVER_NAME=example.com \
    -e SYNAPSE_REPORT_STATS=no \
    matrixdotorg/synapse:latest generate

```

Добавьте в `homeserver.yaml` строку:

```yaml
public_baseurl: "https://matrix.example.com/"
```

Также необходимо прописать данные для подключения к Postgres:

```yaml
database:
  name: psycopg2
  args:
    user: synapse_user
    password: your_strong_password
    dbname: synapse
    host: 1.2.3.4
    port: 5432
    cp_min: 5
    cp_max: 10
    keepalives_idle: 10
    keepalives_interval: 10
    keepalives_count: 3
```

Для работы MatrixRTC звонков необходимо включить federation или openid:

```yaml
listeners:
  - port: 8008
    resources:
    - compress: false
      names:
      - client
#      - federation
      - openid # <--- 
    tls: false
    type: http
    x_forwarded: true
```

Также необходимо включить MSCs (Matrix spec proposals) и указать домен LiveKit сервера.

```yaml
experimental_features:
  msc4222_enabled: true
  msc4354_enabled: true

max_event_delay_duration: 24h
rc_message:
  per_second: 0.5
  burst_count: 30
rc_delayed_event_mgmt:
  per_second: 1
  burst_count: 20

matrix_rtc:
  transports:
  - type: livekit
    livekit_service_url: https://livekit.example.com/livekit/jwt
```

Для работы s3 необходимо прописать настройки хранилища:

```yaml
media_storage_providers:
- module: s3_storage_provider.S3StorageProviderBackend
  store_local: True
  store_remote: True
  store_synchronous: True
  config:
    bucket: <S3_BUCKET_NAME>
    region_name: <S3_REGION_NAME>
    endpoint_url: <S3_LIKE_SERVICE_ENDPOINT_URL>
    access_key_id: <S3_ACCESS_KEY_ID>
    secret_access_key: <S3_SECRET_ACCESS_KEY>
    session_token: <S3_SESSION_TOKEN>
```

**Подробнее:**
- [Документация Synapse](https://element-hq.github.io/synapse/latest/)
- [Образ Synapse на Dockerhub](https://hub.docker.com/r/matrixdotorg/synapse)
- [synapse-s3-storage-provider](https://github.com/matrix-org/synapse-s3-storage-provider)
- [Self-Hosting MatrixRTC](https://github.com/element-hq/element-call/blob/livekit/docs/self_hosting.md)


---


### Caddy

Обратный прокси на базе официального образа [caddy](https://hub.docker.com/_/caddy). Выпускает и обслуживает TLS-сертификаты для каждого поддомена, раздает .well-known для Matrix, а также переадресует на example.com остальные запросы и отдает статику.

По умолчанию Caddy не устанавливает лимит на тело запроса. Для корректной работы установлен лимит в 300MB.

```caddy
request_body {
  max_size 300MB
}
```

Лимит должен совпадать с лимитом в `homeserver.yaml`.

```yaml
max_upload_size: 300M
```

Для корректной работы QUIC (HTTP/3) и WebRTC-трафика LiveKit необходимо увеличить буферы UDP на уровне ядра.

Применить немедленно (до перезагрузки):

```bash
sudo sysctl -w net.core.rmem_max=7500000
sudo sysctl -w net.core.wmem_max=7500000
```

Для сохранения после перезагрузки добавьте в `/etc/sysctl.conf`:

```ini
net.core.rmem_max=7500000
net.core.wmem_max=7500000
```

Значение 7500000 рекомендовано библиотекой [quic-go](https://github.com/quic-go/quic-go/wiki/UDP-Buffer-Sizes), которую использует Caddy.

**Подробнее:**
- [Документация Caddy](https://caddyserver.com/docs/)
- [Hardened image](https://hub.docker.com/hardened-images/catalog/dhi/caddy)


---


### LiveKit

Медиасервер для MatrixRTC. LiveKit не поддерживает переменные окружения для большинства параметров конфигурации, поэтому все настройки прописываются напрямую в `livekit/livekit.yaml`. Скопируйте пример и заполните:

```bash
cp livekit/livekit.yaml.example livekit/livekit.yaml
```

Вместо открытия десятков портов используется UDP mux. Количество портов должно быть **не меньше числа ядер на сервере**:

```yaml
rtc:
  udp_port: 50100-50101
```

Также необходимо открыть эти порты в compose.yaml:

```yaml
livekit:
  ports:
    - "50100-50101:50100-50101/udp"
```

Если вы используете несколько LiveKit серверов, то необходимо использовать Redis.

**Подробнее:**
- [Документация LiveKit](https://docs.livekit.io/transport/self-hosting/deployment/)
- [Пример конфигурации](https://github.com/livekit/livekit/blob/master/config-sample.yaml)


---

### lk-jwt-service

Сервис авторизации для MatrixRTC на базе [официального образа](https://github.com/element-hq/lk-jwt-service/pkgs/container/lk-jwt-service). Выступает посредником между клиентом и LiveKit: выдаёт JWT-токены, подтверждающие право на участие в звонке. Все параметры передаются через переменные окружения в .env — отдельный конфигурационный файл не требуется.
