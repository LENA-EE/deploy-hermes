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

```bash
sudo tee /etc/hermes/model.env >/dev/null <<'EOF'
MODEL=common-qwen
OPENAI_BASE_URL=<LITELLM_URL>
OPENAI_API_KEY=<ключ>
EOF
sudo chmod 600 /etc/hermes/model.env
```

### Расхождения документации с версией v0.13.0

| Симптом | Причина | Правильно |
|---|---|---|
| `Unknown provider 'openai'` / уходит на `openrouter.ai` | в v0.13.0 нет провайдера `openai`; `model:` строкой не понимается | `model.provider: custom`, `base_url` и `api_key` внутри `model:` (так в шаблоне) |
| `Connection error`, хотя `curl` работает | Python в Hermes не доверяет банковскому CA | в `.env` агента: `SSL_CERT_FILE`, `SSL_CERT_DIR`, `REQUESTS_CA_BUNDLE`, `CURL_CA_BUNDLE` → `/etc/ssl/certs/...` (скрипт `hermes-user` пишет сам) |
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

### Пользователь с веб-дашбордом ⏳

```bash
sudo hermes-user add ivanov      # учётка hermes-ivanov, память, порт, веб-пароль, запуск клетки
sudo hermes-user list
```

Требует собранного веб-интерфейса (`hermes_cli/web_dist`): в бандле его нет, готовая сборка —
`dashboard/hermes-dashboard-web-2026.5.7.zip`, установка по `DASHBOARD.md`.
Не запускать `add` для уже настроенного пользователя — перезапишет его конфиг.

## 10. Публикация веб-интерфейса ⏳

Порядок обязателен: **сначала согласование ИБ, потом включение.** Туннели (`ssh -L/-R/-D/-N`)
и любые самостоятельные проброс портов не используются — публикацию делают администраторы.

### 10.1. Что просить у администраторов

Один вход снаружи, независимо от числа пользователей:

> Нужен маршрут до контейнера: проброс порта хоста либо виртуальный хост в корпоративном прокси.
> На этом порту в контейнере будет **nginx с авторизацией**, он направляет каждого пользователя
> в его экземпляр. Нужны WebSocket и таймаут ~3600 с. Уточнить, кто терминирует HTTPS.

Пример полученного маршрута (сентябрь 2026):

```
https://hermes_agent_drospr.bc-dev2.int.gazprombank.ru  →  http://<IP контейнера>:9121
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

`/etc/nginx/conf.d/hermes.conf` (публикация через прокси, TLS снаружи):

```nginx
map $remote_user $hermes_port {
    include /etc/nginx/hermes-users.map;   # строки вида: tulkin 9123;  — генерирует hermes-user
    default 0;
}

server {
    listen 9121;                                   # порт, на который смотрит корпоративный прокси
    server_name <домен сервиса>;

    auth_basic           "Hermes";
    auth_basic_user_file /etc/nginx/hermes.htpasswd;

    location / {
        if ($hermes_port = 0) { return 403; }      # логин без сопоставленного экземпляра — отказ
        proxy_pass http://127.0.0.1:$hermes_port;

        proxy_http_version 1.1;
        proxy_set_header Upgrade    $http_upgrade; # чат работает по websocket
        proxy_set_header Connection "upgrade";
        proxy_set_header Host       $host;
        proxy_set_header X-Real-IP  $remote_addr;
        proxy_set_header Authorization "";         # пароль nginx дальше не передаётся
        proxy_read_timeout 3600s;                  # агент отвечает долго
        proxy_send_timeout 3600s;
    }
}
```

### 10.4. Включение и проверка

```bash
sudo nginx -t && sudo systemctl enable --now nginx          # проверить конфиг и поднять nginx
sudo ss -tlnp | grep -E ':912[0-9]'                         # ждём: nginx на 9121, экземпляры на 127.0.0.1:9122+
sudo systemctl start hermes-dashboard@agent                 # запускать экземпляры после согласования
```

Проверки со стороны пользователя: вход по домену требует логин и пароль; после входа открывается
свой экземпляр; чат отвечает потоком (значит, websocket проходит); чужой логин ведёт в чужой
экземпляр только с чужим паролем.

### 10.5. Дальше

- Проверка изоляции атаками — часть 4 проектного плана.
- Обновление версии Hermes — переключением ссылки `/opt/hermes` (см. `DASHBOARD.md`, шаг 1).
