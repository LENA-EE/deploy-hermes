# Установка Hermes в контейнер — инструкция для следующих

Проверено на: LXC, Debian 13, systemd 257, офлайн-бандл Hermes Agent **v0.13.0**, сентябрь 2026.
Файлы, на которые ссылается инструкция, лежат в папке `deploy/` этого репозитория.

Метки: ✅ — проверено на практике, ⏳ — написано, но ещё не прогонялось целиком.

---

## 0. Что должно быть до начала

| Что | Как проверить |
|---|---|
| Учётка с `sudo` | `sudo -l` |
| PID 1 — systemd | `cat /proc/1/comm` → `systemd` |
| Нет docker-сокета | `ls /var/run/docker.sock` → `No such file` |
| Песочница systemd разрешена в LXC | тест ниже ✅ |
| Доступ к LiteLLM и ключ | `curl -H "Authorization: Bearer $KEY" <LITELLM>/v1/models` |
| Банковское зеркало пакетов (Nexus) | `apt-cache policy nginx` → есть `Candidate` |

Тест песочницы (`systemd-run` в контейнере не работает — нет D-Bus, поэтому через юнит-файл):

```bash
sudo tee /etc/systemd/system/sandbox-test.service >/dev/null <<'EOF'
[Service]
Type=oneshot
User=nobody
ProtectHome=tmpfs
TemporaryFileSystem=/srv:ro
ProtectProc=invisible
PrivateTmp=yes
ProtectSystem=strict
ExecStart=/bin/sh -c 'ls /home; ls /srv; echo SANDBOX_OK'
EOF
sudo systemctl daemon-reload && sudo systemctl start sandbox-test
sudo journalctl -u sandbox-test -n 10 --no-pager -o cat      # ждём: пусто, пусто, SANDBOX_OK
sudo rm /etc/systemd/system/sandbox-test.service && sudo systemctl daemon-reload
```

## 1. Пакеты ✅

```bash
sudo apt-get update
sudo apt-get install -y nftables nginx curl
sudo systemctl disable --now nginx        # пока не настроен — не держать открытый порт 80
```

## 2. Бандл ✅

Залить архив по SFTP в свой хомяк и распаковать туда же (хомяк `700` — агенты его не видят).

```bash
mkdir -p ~/hermes-bundle && tar -xzf ~/hermes-offline-wsl2.tar.gz -C ~/hermes-bundle
B=~/hermes-bundle/bundle
cat $B/MANIFEST.txt                       # записать версию
```

## 3. Банковский сертификат ✅

```bash
ls /usr/local/share/ca-certificates/      # есть gpb-root.crt — шаг пропустить
sudo HOME=/root bash $B/skills/gpb-ssl-setup/setup-ssl.sh   # если нет
```

## 4. Установка кода в `/opt/hermes` ✅

**Важно:** шаг с `debs/` пропускается. На Debian он либо ставит пакеты, собранные под Ubuntu, поверх
системных, либо при сбое дописывает в `.bashrc` `LD_LIBRARY_PATH`, нарушая работу `ls`, `git`, `apt`.
Пакеты требуются только браузерным и голосовым инструментам, которые отключены.

```bash
mv $B/debs $B/debs.disabled
sudo install -d -m 755 /opt/hermes
sudo env HOME=/opt/hermes/.install-home \
         HERMES_HOME=/opt/hermes/.install-home/.hermes \
         HERMES_INSTALL_DIR=/opt/hermes/hermes-agent \
         HERMES_TOOLS_DIR=/opt/hermes/tools \
         PLAYWRIGHT_BROWSERS_PATH=/opt/hermes/ms-playwright \
     bash $B/scripts/install-on-wsl.sh 2>&1 | tee ~/hermes-installer.log
sudo chown -R root:root /opt/hermes
sudo chmod -R go-w,a+rX /opt/hermes
sudo grep -n LD_LIBRARY_PATH /root/.bashrc ~/.bashrc /home/*/.bashrc   # ждём пусто
```

Предупреждения `doesn't look like WSL2` и `agent-browser CLI failed self-test` — нормально.

## 5. Запускалка ✅

```bash
sudo install -d -m 755 /etc/hermes
sudo install -m 755 deploy/hermes-run /usr/local/bin/
sudo tee /etc/hermes/hermes.env >/dev/null <<'EOF'
HERMES_PYTHON=/opt/hermes/hermes-agent/venv/bin/python
HERMES_PATH_PREPEND=/opt/hermes/tools/node/bin:/opt/hermes/tools
PLAYWRIGHT_BROWSERS_PATH=/opt/hermes/ms-playwright
HERMES_GIT_BASH_PATH=/bin/bash
EOF
sudo chmod 644 /etc/hermes/hermes.env

sudo -u nobody env HOME=/tmp hermes-run --version    # ждём: Hermes Agent v0.13.0
sudo -u nobody touch /opt/hermes/x                   # ждём: Permission denied
```

## 6. Шаблоны платформы ✅

```bash
sudo install -m 644 deploy/hermes-dashboard@.service /etc/systemd/system/
sudo install -m 644 deploy/hermes.slice              /etc/systemd/system/
sudo install -m 644 deploy/config.template.yaml      /etc/hermes/
sudo install -m 755 deploy/hermes-user               /usr/local/sbin/
sudo install -d -m 755 /etc/hermes/users /srv/hermes
sudo systemctl daemon-reload
```

## 7. Настройки модели ✅

`/etc/hermes/model.env` — источник настроек для **новых** пользователей: `hermes-user add`
берёт отсюда модель, адрес и ключ. Без этого файла команда откажет:
`hermes-user: нет /etc/hermes/model.env`.

Семь строк: три про модель и четыре про банковский сертификат. Скрипт копирует в `.env`
каждого нового пользователя все строки, кроме `MODEL=`, — поэтому SSL-переменные, которые
Hermes иначе не получит, задаются здесь же один раз, а не дописываются потом руками.

```bash
sudo tee /etc/hermes/model.env >/dev/null <<'EOF'
MODEL=common-qwen
BASE_URL=<LITELLM_URL>
API_KEY=<ключ>
SSL_CERT_FILE=/etc/ssl/certs/ca-certificates.crt
SSL_CERT_DIR=/etc/ssl/certs
REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt
CURL_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt
EOF
sudo chmod 600 /etc/hermes/model.env      # rw-------: в файле ключ, читает только root
sudo grep -c . /etc/hermes/model.env       # ждём 7 непустых строк
```

Если первый пользователь уже настроен вручную, значения можно взять из его конфига,
не печатая ключ на экран:

```bash
sudo tee /etc/hermes/model.env >/dev/null <<EOF
MODEL=common-qwen
BASE_URL=$(sudo sed -n 's/^ *base_url: *//p' /home/hermes-agent/.hermes/config.yaml)
API_KEY=$(sudo sed -n 's/^ *api_key: *//p' /home/hermes-agent/.hermes/config.yaml)
SSL_CERT_FILE=/etc/ssl/certs/ca-certificates.crt
SSL_CERT_DIR=/etc/ssl/certs
REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt
CURL_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt
EOF
sudo chmod 600 /etc/hermes/model.env
```

### 7.1. Шаблон конфига: три условия ✅ (проверено 2026-09-25)

`hermes-user add` собирает `config.yaml` пользователя из `/etc/hermes/config.template.yaml`,
подставляя значения из `model.env`. Всё, что заведено раньше, при правке шаблона **не меняется**:
`hermes-user regen` пересобирает только карту nginx и правила nftables, конфиги пользователей — нет.

Три условия, без которых новый пользователь окажется нерабочим:

| Условие | Если нарушено |
|---|---|
| В шаблоне `model:` — блок с `provider: custom`, `name`, `default`, `base_url`, `api_key` | `AuthError: No inference provider configured` |
| В скрипте есть подстановка всех четырёх плейсхолдеров: `@WORKDIR@`, `@MODEL@`, `@BASE_URL@`, `@API_KEY@` | `Failed to parse config.yaml: found character '@'` → молчаливый откат к умолчаниям и тот же `AuthError` |
| В `model.env` есть четыре строки `SSL_*`/`*_CA_BUNDLE` | `API call failed after 3 retries: Connection error` при работающем `curl` |

Проверка шаблона и скрипта перед первым заведением:

```bash
sudo grep -c '@WORKDIR@\|@MODEL@\|@BASE_URL@\|@API_KEY@' /usr/local/sbin/hermes-user   # ждём 1: все в одной строке sed
sudo grep -A5 '^model:' /etc/hermes/config.template.yaml    # ждём блок с provider: custom
```

Имена плейсхолдеров не должны встречаться в комментариях шаблона: подстановка идёт по всему файлу,
и ключ окажется в готовом конфиге ещё и в комментарии.

Приёмка на одноразовом пользователе — проверка того, что следующее заведение пройдёт без ручных правок:

```bash
sudo hermes-user add testuser                                      # пароль не короче 12 символов
sudo grep -q '@' /home/hermes-testuser/.hermes/config.yaml   && echo 'ПЛОХО: плейсхолдеры не подставлены' || echo 'ок: плейсхолдеры подставлены'
sudo -iu hermes-testuser hermes-run -z "Ответь одним словом: работает"   # ждём ответ модели
sudo hermes-user purge testuser                                    # убрать за собой
```

Если тестовый пользователь отвечает без ручных правок, заведение сотрудника сводится к одной команде.

### Расхождения документации с версией v0.13.0

| Симптом | Причина | Правильно |
|---|---|---|
| `Unknown provider 'openai'` / уходит на `openrouter.ai` | в v0.13.0 нет провайдера `openai`; `model:` строкой не понимается | `model.provider: custom`, `base_url` и `api_key` внутри `model:` (так в шаблоне) |
| `Connection error`, хотя `curl` работает | Python в Hermes не доверяет банковскому CA | четыре строки `SSL_*`/`*_CA_BUNDLE` в `/etc/hermes/model.env` → скрипт копирует их в `.env` каждого пользователя |
| Настройки «не применяются» | YAML сломан → Hermes молча берёт умолчания | `hermes-run logs errors --since 10m`; конфиг править целиком, не `sed` по номерам строк |

## 8. Файрвол ⏳

```bash
sudo install -d -m 755 /etc/nftables.d
grep -q 'nftables.d' /etc/nftables.conf || echo 'include "/etc/nftables.d/*.nft"' | sudo tee -a /etc/nftables.conf
sudo nft -c -f /etc/nftables.conf && echo CONF_OK
sudo systemctl enable nftables
```

## 9. Пользователи

### Консольный пользователь вручную ✅

Проверенная последовательность (пример для `hermes-agent`):

```bash
U=hermes-agent; W=/srv/hermes/agent
sudo install -d -m 700 -o $U -g $U /home/$U/.hermes $W
sudo tee /home/$U/.hermes/.env >/dev/null <<EOF
OPENAI_BASE_URL=<LITELLM_URL>
OPENAI_API_KEY=<ключ>
HERMES_WRITE_SAFE_ROOT=$W
SSL_CERT_FILE=/etc/ssl/certs/ca-certificates.crt
SSL_CERT_DIR=/etc/ssl/certs
REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt
CURL_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt
EOF
sed -e "s|@WORKDIR@|$W|" -e "s|@MODEL@|common-qwen|g" \
    -e "s|@BASE_URL@|<LITELLM_URL>|" -e "s|@API_KEY@|<ключ>|" \
    /etc/hermes/config.template.yaml | sudo tee /home/$U/.hermes/config.yaml >/dev/null
sudo chown $U:$U /home/$U/.hermes/.env /home/$U/.hermes/config.yaml
sudo chmod 600  /home/$U/.hermes/.env /home/$U/.hermes/config.yaml
sudo -iu $U hermes-run -z "Ответь одним словом: работает"
```

### Пользователь с веб-дашбордом ✅ (проверено 2026-09-25)

Требуются: собранный веб-интерфейс (`DASHBOARD.md`), заполненный `/etc/hermes/model.env` (шаг 7)
и настроенный nginx (шаг 10).

```bash
sudo hermes-user add ivanov      # учётка hermes-ivanov, память, рабочая папка, свободный порт,
                                 # веб-пароль (спросит дважды, ≥12 символов), запись для nginx,
                                 # правила nftables, запуск клетки
sudo hermes-user list            # ждём: ivanov  9123  yes  active
```

Проверка нового пользователя, включая изоляцию:

```bash
sudo -iu hermes-ivanov hermes-run -z "Ответь одним словом: работает"                  # его модель отвечает
curl -s -o /dev/null -w '%{http_code}' -u ivanov http://<IP контейнера>:9121; echo    # ждём 200
sudo -iu hermes-ivanov cat /home/hermes-agent/.hermes/config.yaml                     # ждём Permission denied
```

Последняя команда — практическое подтверждение, что пользователи не видят файлы друг друга.

Не запускать `add` для уже настроенного пользователя: команда перезапишет его конфиг.
Прочие операции: `hermes-user passwd <имя>`, `disable <имя>`, `purge <имя>`, `list`.

## 10. Публикация веб-интерфейса ✅ (проверено 2026-09-25)

Порядок обязателен: **сначала согласование ИБ, потом включение.** Сетевой доступ к веб-интерфейсу
открывают администраторы по согласованной заявке; самостоятельно сетевые проходы не создаются.

### 10.1. Что просить у администраторов

Один вход снаружи, независимо от числа пользователей:

> Нужен маршрут до контейнера: проброс порта хоста либо виртуальный хост в корпоративном прокси.
> На этом порту в контейнере будет **nginx с авторизацией**, он направляет каждого пользователя
> в его экземпляр. Нужны WebSocket и таймаут ~3600 с. Уточнить, кто терминирует HTTPS.

Пример полученного маршрута (сентябрь 2026):

```
https://hermes.<домен-сервиса>  →  http://<IP контейнера>:9121
```

HTTPS терминирует корпоративный прокси, внутрь идёт обычный HTTP — сертификат в контейнере не нужен.

### 10.2. Раскладка портов

⚠️ На опубликованном порту должен слушать **nginx**, а не дашборд: иначе вход открыт без пароля.

```
прокси → <IP контейнера>:9121 = nginx (basic auth, маршрутизация по логину)
                                  ├─ 127.0.0.1:9122  экземпляр agent
                                  ├─ 127.0.0.1:9123  экземпляр tulkin
                                  └─ 127.0.0.1:912x  …
```

Экземпляры пользователей слушают только loopback; наружу выходит единственный порт nginx.
`hermes-user` выдаёт номера из диапазона 9121–9199 — занятый nginx номер занять в `/etc/hermes/users/`
служебной записью либо начинать нумерацию с 9122.

### 10.3. Конфигурация nginx

Проверено 2026-09-25. Четыре неочевидных места отмечены комментариями — без них
страница открывается, но чат не работает.
Пятое место — `client_max_body_size`: без него не загружаются файлы больше 1 МБ
(добавлено 2026-10-13, ⏳ в контейнере ещё не проверено).

`/etc/nginx/conf.d/hermes.conf` (публикация через корпоративный прокси, TLS снаружи):

```nginx
map $remote_user $hermes_port {
    include /etc/nginx/hermes-users.map;   # строки вида: agent 9122;  — генерирует hermes-user
    default 1;                             # неизвестный логин → порт, где никто не слушает
}

server {
    listen <IP-контейнера>:9121;              # адрес контейнера, на него смотрит корпоративный прокси
    server_name hermes.<домен-сервиса>;

    auth_basic           "Hermes";         # сначала логин и пароль
    auth_basic_user_file /etc/nginx/hermes.htpasswd;

    # (5) размер загружаемого файла. По умолчанию nginx пропускает только 1 МБ,
    #     файлы больше отбиваются с 413 ещё до Hermes. Цель — файлы до 10 МБ,
    #     12m — с запасом на служебную часть запроса.
    client_max_body_size 12m;

    location / {
        proxy_pass http://127.0.0.1:$hermes_port;   # маршрут по логину

        proxy_http_version 1.1;
        proxy_set_header Upgrade    $http_upgrade;  # websocket: чат работает только через него
        proxy_set_header Connection "upgrade";

        # (1) дашборд проверяет Host и принимает только имя, на которое привязан.
        #     Он слушает 127.0.0.1 → передаём именно это, иначе 400 Invalid Host header.
        proxy_set_header Host "127.0.0.1";

        # (2) uvicorn по умолчанию доверяет заголовкам прокси и подменяет адрес клиента
        #     на значение X-Forwarded-For. Дашборд затем считает websocket-клиента
        #     неподходящим (ожидает локальный адрес) и отвечает 403 на рукопожатие.
        #     Пустое значение = заголовок не передаётся.
        proxy_set_header X-Real-IP         "";
        proxy_set_header X-Forwarded-For   "";
        proxy_set_header X-Forwarded-Proto "";

        # (3) пароль nginx дальше не передаётся: у дашборда своя проверка токена
        proxy_set_header Authorization "";

        proxy_read_timeout 3600s;                   # (4) агент отвечает долго, минута не годится
        proxy_send_timeout 3600s;

        proxy_intercept_errors on;
        error_page 502 504 = @nouser;
    }

    location @nouser {
        return 403 "No Hermes instance for this login
";
    }
}
```

⚠️ Не использовать конструкцию `if ($hermes_port = 0) { return 403; }`: в nginx `if`
выполняется раньше проверки пароля, поэтому запрос отбивается до авторизации (403 вместо 401),
а `proxy_pass` получает пустой порт (400). Вместо неё — `default 1` в карте и `error_page`.

### 10.4. Включение: последовательность ✅

Порядок важен: сначала экземпляр освобождает опубликованный порт, затем на него встаёт nginx.
Пример для контейнера с адресом `<IP-контейнера>`, опубликован порт 9121, первый пользователь — `agent`.

```bash
# 0) адрес контейнера — он должен совпадать с тем, куда направлен корпоративный прокси
ip -4 addr show | grep inet                                             # ждём адрес вида <IP-контейнера>

# 1) экземпляр пользователя переезжает с опубликованного порта на внутренний
cat /etc/hermes/users/agent.conf                                        # что сейчас
sudo sed -i 's/^PORT=9121$/PORT=9122/' /etc/hermes/users/agent.conf     # сменить номер порта
sudo systemctl restart hermes-dashboard@agent                           # применить
sudo ss -tlnp | grep 9122                                               # ждём 127.0.0.1:9122

# 2) веб-пароль пользователя (попадает в /etc/nginx/hermes.htpasswd в виде хеша)
sudo hermes-user passwd agent

# 3) таблица «логин → порт» и правила изоляции портов
sudo hermes-user regen
cat /etc/nginx/hermes-users.map                                         # ждём строку: agent 9122;

# 4) конфиг nginx из 10.3 (listen <IP контейнера>:9121), затем
sudo rm -f /etc/nginx/sites-enabled/default                             # убрать стандартную заглушку
sudo nginx -t && sudo systemctl enable --now nginx                      # проверить конфиг и запустить

# 5) проверки ДО обращения из браузера (<IP> — адрес контейнера из шага 0)
sudo ss -tlnp | grep -E ':912[0-9]'                                     # nginx на <IP>:9121, экземпляры только на 127.0.0.1
curl -s -o /dev/null -w '%{http_code}' http://<IP>:9121; echo           # без пароля: ждём 401
curl -s -o /dev/null -w '%{http_code}' -u agent http://<IP>:9121; echo  # пароль curl спросит отдельно: ждём 200
```

Пароль не писать в командной строке (`-u agent:пароль`): он осядет в истории команд и будет
виден в списке процессов. Форма `-u agent` запрашивает его отдельно и нигде не сохраняет.

Как читать результат:

| Ответ | Значение |
|---|---|
| `401` без пароля | вход закрыт, авторизация работает |
| `200` без пароля | ⚠️ наружу выставлен сам экземпляр, а не nginx: `sudo systemctl stop 'hermes-dashboard@*'` и вернуться к шагу 1 |
| `200` с паролем | схема собрана верно, можно открывать домен в браузере |
| `502` / `503` с паролем | nginx работает, но не достучался до экземпляра: проверить, запущен ли он и слушает ли свой порт |
| `000` | никто не слушает адрес: nginx не запущен или слушает не тот адрес |

Только после `401`/`200` — открывать домен в браузере. Следующие пользователи заводятся
командой `sudo hermes-user add <имя>`: порт, пароль и строка в таблице создаются автоматически.

Диагностика: `sudo nginx -t`, `sudo tail -20 /var/log/nginx/error.log`,
`sudo journalctl -u hermes-dashboard@<имя> -n 20 --no-pager`.

### 10.5. Диагностика веб-доступа

| Симптом | Причина | Решение |
|---|---|---|
| `403` без запроса пароля | в конфиге есть `if (...) { return 403; }` — срабатывает до авторизации | убрать `if`, использовать `default 1` + `error_page` |
| `400`, в теле `Invalid Host header` | дашборд принимает только имя, на которое привязан | `proxy_set_header Host "127.0.0.1";` |
| Страница открывается, чат молчит; в консоли браузера `WebSocket … 403` | uvicorn подменил адрес клиента по `X-Forwarded-For`, дашборд отклонил websocket | очистить `X-Real-IP`, `X-Forwarded-For`, `X-Forwarded-Proto` |
| `WebSocket … 403`, токен в адресе старый | токен сессии создаётся заново при каждом запуске дашборда | перезапустить дашборд, открыть страницу в приватном окне |
| Ответ обрывается через минуту | таймаут на корпоративном прокси | попросить администраторов `proxy_read_timeout 3600s` и поддержку WebSocket |
| Файл больше 1 МБ не загружается, `413 Request Entity Too Large` | не задан `client_max_body_size`, у nginx по умолчанию 1 МБ | `client_max_body_size 12m;` в `server`, затем `sudo nginx -t && sudo systemctl reload nginx`. Если 413 осталась — лимит на корпоративном прокси, просить администраторов |
| `model catalog fetch failed … Name or service not known` | дашборд тянет справочник моделей с сайта проекта, интернета нет | безвредно: модель задаётся в `config.yaml`, список в интерфейсе останется пустым |

Полезные команды:

```bash
sudo grep -E "api/(ws|events|pty)" /var/log/nginx/access.log | tail -3   # ждём код 101 — websocket открылся
sudo tail -20 /var/log/nginx/error.log                                   # строки sched_setaffinity в LXC безвредны
ps -eo cmd | grep "hermes_cli.main dashboard" | grep -v grep             # проверить наличие --tui: без него чат отключён
sudo -iu hermes-<имя> hermes-run -z "Ответь: работает"                   # модель отвечает в обход веба
```

### 10.5. Дальше

- Проверка изоляции атаками — часть 4 проектного плана.
- Обновление версии Hermes — переключением ссылки `/opt/hermes` (см. `DASHBOARD.md`, шаг 1).
