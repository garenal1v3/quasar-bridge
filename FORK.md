# Форк quasar-bridge для «Станции»

Это форк [skrashevich/quasar-bridge](https://github.com/skrashevich/quasar-bridge),
серверная часть проекта «Станция»: своя Яндекс Станция ходит через свой бэкенд.

## Зачем форк

Архитектура в три слоя:

1. **YAM-пакет `station-link` на колонке** — на рутованной колонке переключает
   речевой тракт (`aliced.uniProxyUrl`) на прокси и отвечает за пейринг.
2. **Этот прокси** — принимает нативный WebSocket-протокол Quasar/Uniproxy.
   По умолчанию это **полный passthrough в Яндекс**: весь трафик колонки без
   изменений уходит в настоящий `wss://uniproxy.alice.yandex.net/uni.ws`.
   Здесь же точки подмены STT/LLM/TTS/музыки на следующих фазах.
3. **Приватные провайдеры** (LLM, музыка) — подключаются позже, за прокси.

Фаза 0 — только passthrough: штатный режим `observe`, логика маршрутизации
не менялась.

## Фаза 0: сборка и запуск

### Сборка

Нужен Go 1.27.3+ (см. `go.mod`):

```sh
go build -trimpath -o bin/quasar-bridge ./cmd/quasar-bridge
```

Если Go на хосте нет, можно собрать в контейнере, ничего не ставя в систему:

```sh
docker run --rm -u "$(id -u):$(id -g)" \
  -e HOME=/tmp -e GOCACHE=/tmp/gocache -e GOMODCACHE=/tmp/gomod \
  -v "$PWD":/src -w /src golang:1.27.3 \
  go build -trimpath -o bin/quasar-bridge ./cmd/quasar-bridge
```

### Конфиг

`config.json` в `.gitignore`, в репозиторий не коммитится. Создаётся из
`config.observe.example.json`; отличия: слушаем всю LAN и **без TLS**
(в фазе 0 сервер и колонка в одной LAN, пиннинга сертификатов на прошивке нет,
`ws://` работает):

```sh
umask 077
cp config.observe.example.json config.json
# затем поменять: listen → "0.0.0.0:8443", tls_cert и tls_key → ""
```

Итоговые ключевые поля:

```json
{
  "listen": "0.0.0.0:8443",
  "upstream": "wss://uniproxy.alice.yandex.net/uni.ws",
  "mode": "observe",
  "tls_cert": "",
  "tls_key": "",
  "allowed_ips": []
}
```

Остальные разделы (`conversation_log`, `assistant_tools`) — как в примере,
выключены.

### Запуск и проверка

```sh
./bin/quasar-bridge -config config.json
```

В другом терминале:

```sh
curl -s http://127.0.0.1:8443/healthz
```

Ожидается JSON со `"status":"ok"` и `"mode":"observe"`.

Колонка направляется на `ws://SERVER_IP:8443/uni.ws` (это делает YAM-пакет
`station-link`, либо вручную — см. README, раздел про подключение колонки).
Пустой `allowed_ips` пускает любой адрес; порт стоит держать закрытым снаружи LAN.
