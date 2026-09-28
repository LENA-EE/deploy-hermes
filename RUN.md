# Запуск и повседневная работа с Hermes

Для установленного контейнера (см. `INSTALL.md`). Все команды — со своей учётки через `sudo`.

> **Никогда не запускать Hermes под своей учёткой.** У неё sudo без пароля — агент получил бы root.
> Только `sudo -iu hermes-<имя> hermes-run ...` (с `-i`: иначе Hermes прочитает не тот `~/.hermes`).

## Поговорить с агентом (консоль)

```bash
sudo -iu hermes-agent hermes-run                                   # интерактивный чат; выход — /exit или Ctrl-D
sudo -iu hermes-agent hermes-run -c                                # продолжить последний разговор
sudo -iu hermes-agent hermes-run -z "Ответь одним словом: работает" # один вопрос — для проверки
```

`-z` пропускает подтверждения опасных команд — только для проверок, не для работы.

## Проверить, что всё живо

```bash
sudo -iu hermes-agent hermes-run doctor                  # самодиагностика
sudo -iu hermes-agent hermes-run logs errors --since 1h  # ошибки за час — первое, что смотреть
sudo -iu hermes-agent hermes-run logs -f                 # журнал в реальном времени, выход Ctrl-C
sudo -iu hermes-agent cat /home/hermes-agent/.hermes/config.yaml   # что реально в конфиге
```

## Веб-дашборды (когда будут)

```bash
sudo hermes-user list                                    # кто заведён, порт, работает ли
sudo systemctl status hermes-dashboard@ivanov            # состояние одного
sudo systemctl restart hermes-dashboard@ivanov           # перезапуск после правки его конфига
sudo journalctl -u hermes-dashboard@ivanov -n 50         # его журнал
sudo systemctl stop 'hermes-dashboard@*'                 # аварийно остановить всех
```

## Доступ к веб-интерфейсу

Вход один, по домену сервиса: корпоративный прокси → `<IP контейнера>:9121` → nginx → экземпляр
пользователя (`127.0.0.1:9122` и далее). Пользователь вводит логин и пароль, созданные `hermes-user`.

Самостоятельный проброс сетевого доступа не используется: публикацию выполняют администраторы.
Проверка работоспособности из консоли контейнера:

```bash
curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:9122; echo    # экземпляр отвечает: ждём 200
curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:9121; echo    # nginx без пароля: ждём 401
sudo ss -tlnp | grep -E ':912[0-9]'                                    # nginx на 9121, экземпляры только на 127.0.0.1
sudo nginx -t                                                          # конфиг nginx корректен
sudo tail -5 /var/log/nginx/error.log                                  # ошибки входа и проксирования
```

## Пользователи

Перед первым `add` должен существовать `/etc/hermes/model.env` — семь строк: `MODEL`, `BASE_URL`,
`API_KEY` и четыре `SSL_*`/`*_CA_BUNDLE` (см. `INSTALL.md`, шаг 7). Без файла команда откажет
с сообщением «нет /etc/hermes/model.env».

Конфиги уже заведённых пользователей при правке шаблона или `model.env` **не обновляются**:
`regen` пересобирает только карту nginx и правила nftables. Проще всего пересоздать: `purge` + `add`.

```bash
sudo hermes-user add ivanov       # завести
sudo hermes-user passwd ivanov    # сменить веб-пароль
sudo hermes-user disable ivanov   # закрыть вход, данные сохранить
sudo hermes-user purge ivanov     # удалить всё (спросит подтверждение)
```

## Раз в неделю

```bash
df -h /                                                   # диск без квоты — держать ниже 80%
sudo du -sh /home/hermes-* /srv/hermes/* | sort -h        # кто растёт
sudo systemd-cgtop -1 /hermes.slice                       # кто ест память и CPU
```

## Частые вопросы пользователей

| Вопрос | Ответ до обновления | После v0.21.0 |
|---|---|---|
| Как загрузить файл агенту | возможности нет: копировать текстом в чат либо просить администратора положить файл в рабочую папку | страница **Files**: загрузка в браузере |
| Как забрать файл у агента | вывести содержимое в чат и скопировать; файлы выгружает администратор | страница **Files**: скачивание |
| В чате не выделяется текст | вкладка чата — встроенный терминал, мышь перехвачена. Выделять с зажатым **Shift**; либо смотреть переписку на странице **Сессии**, там обычный текст | так же: чат остаётся терминалом |
| Не выбираются модели | подключена одна модель — выбирать не из чего. Список доступных проверяется запросом `/v1/models` к сервису модели | выбор появится, если моделей несколько |
| Можно ли вставить скриншот | нет | да, вставкой из буфера или перетаскиванием |

## Если что-то не так

| Симптом | Где смотреть / что делать |
|---|---|
| `Failed to parse config.yaml: found character '@'` | в скрипте нет подстановки `@BASE_URL@`/`@API_KEY@` — см. `INSTALL.md`, шаг 7.1 |
| `Unknown provider` | в `config.yaml` должно быть `model.provider: custom` |
| `Connection error` | в `.env` агента четыре строки `SSL_*` / `*_CA_BUNDLE`; `logs errors` |
| Настройки не действуют | `logs errors` → «Falling back to default config» = сломан YAML |
| Hermes читает не тот конфиг | запускать с `sudo -iu` (буква `i`); `config show` → раздел Paths |
| `~` указывает не туда | в `sudo -iu X команда ~/...` тильду разворачивает **твой** shell — писать полный путь |
| После правки YAML всё сломалось | перезаписать файл целиком (`sudo tee`), не удалять строки по номерам |
| `hermes-user: пароль короче 12 символов` | `add` прерывается на полпути: учётка и порт есть, сервис `inactive` — доделать `passwd`, затем `systemctl enable --now hermes-dashboard@<имя>` |
| `hermes-user: нет /etc/hermes/model.env` | не создан файл с настройками модели для новых пользователей — см. `INSTALL.md`, шаг 7 |
| Веб открывается, чат молчит, в консоли браузера `WebSocket … 403` | nginx передаёт `X-Forwarded-For`, дашборд считает клиента нелокальным — очистить `X-Real-IP`, `X-Forwarded-For`, `X-Forwarded-Proto` в `/etc/nginx/conf.d/hermes.conf` |
| `400 Invalid Host header` | дашборд принимает только имя, на которое привязан — `proxy_set_header Host "127.0.0.1";` |

Порядок диагностики: `logs errors` → `doctor` → проверка путей в `config show` → гипотезы.
