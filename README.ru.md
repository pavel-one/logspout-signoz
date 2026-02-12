# logspout-signoz

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

Русская версия | [English](README.md)

Минималистичный адаптер для [logspout](https://github.com/gliderlabs/logspout) для отправки уведомлений в [SigNoz](https://signoz.io/) через http(s) endpoint.

### Зачем мне это нужно?

Допустим, вы запускаете свое приложение с помощью docker или docker compose. Вы хотите отправлять логи в
SigNoz, тогда вы можете использовать этот адаптер для отправки логов в SigNoz.

### Какие возможности он предоставляет?

1. Прямая отправка на http endpoint SigNoz. Таким образом, этот адаптер может отправлять более подробные логи.
1. Автоматическое определение имени сервиса, поэтому специальная конфигурация не требуется.
   1. Для JSON логов берет имя из JSON поля service.
   1. В противном случае берет имя сервиса из имени сервиса docker-compose.
   1. В противном случае использует имя docker образа в качестве имени сервиса
1. Автоматическое определение имени окружения, поэтому специальная конфигурация не требуется
   1. Для JSON логов берет имя из JSON поля env.
   1. В противном случае берет env из переменной окружения logspout-signoz ENV.
1. Автоматический парсинг JSON логов.
   1. Сопоставляет известные JSON атрибуты логов с соответствующими полями Signoz log payload. Например, `level` с `SeverityText` и т.д.
   1. Упаковывает другие JSON атрибуты в ключ attributes Signoz log payload.

### Как использовать?

Сначала включите httpreciver, добавив следующее в `otel-collector-config.yaml`

1. Добавьте `httplogreceiver/json` в секцию `receivers`
1. Добавьте `httplogreceiver/json` в `service.pipelines.logs.receivers`

```yaml
receivers:
  httplogreceiver/json:
    endpoint: 0.0.0.0:8082
    source: json

...

service:
   pipelines:
      logs:
         receivers: [otlp, tcplog/docker, httplogreceiver/json]
         processors: [batch]
         exporters: [clickhouselogsexporter]
```

Откройте порт 8082 в вашем otel-collector контейнере следующим образом:

```yaml
services:
   otel-collector:
   image: signoz/signoz-otel-collector:${OTELCOL_TAG:-0.102.10}
   container_name: signoz-otel-collector
   ports:
      - "8082:8082" # Логи SigNoz
```

Затем запустите контейнер logspout-signoz следующей командой: 
(Запустите это на каждом узле, где вы хотите собирать логи)

```bash
docker run -d \
        --volume=/var/run/docker.sock:/var/run/docker.sock \
        -e 'SIGNOZ_LOG_ENDPOINT=http://1.2.3.4:8082' \
        -e 'ENV=prod' \
        orendat/logspout-signoz \
        signoz://localhost:8082
```

### Параметры конфигурации

Вы можете использовать следующие переменные окружения для настройки адаптера:

- `SIGNOZ_LOG_ENDPOINT`: URL endpoint'а логов SigNoz. По умолчанию: `http://localhost:8082`
- `ENV`: Имя окружения.
- `DISABLE_JSON_PARSE`: Любое строковое значение отключит парсинг JSON и будет отправлять JSON лог как есть.
- `DISABLE_LOG_LEVEL_STRING_MATCH`: Для не-JSON логов этот адаптер пытается определить уровень логирования, ища строки
   "ERROR", "INFO" и т.д. и сопоставляя их с уровнем серьезности логов Signoz. Присвоение любого строкового значения этой переменной окружения отключит 
   определение уровня логирования.


### Как собрать и запустить?

Следуйте инструкциям для сборки собственного [образа logspout](https://github.com/gliderlabs/logspout/tree/master/custom), включающего этот модуль.
Вкратце, скопируйте содержимое папки `custom` и добавьте следующую строку импорта выше остальных в `modules.go`:
```go
package main

import (
  _ "github.com/pavel-one/logspout-signoz/signoz"
  // ...
)
```

Если вы хотите выбрать определенную версию, создайте следующий `Dockerfile`:
```
ARG VERSION
FROM gliderlabs/logspout:$VERSION

ONBUILD COPY ./build.sh /src/build.sh
ONBUILD COPY ./modules.go /src/modules.go
```

Затем соберите образ командой: `docker build --no-cache --pull --force-rm --build-arg VERSION=v3.2.14 -f dockerfile -t logspout:v3.2.14 .`


## Параметры конфигурации Logspout

Вы можете использовать стандартные фильтры logspout для фильтрации имен контейнеров и типов вывода:

