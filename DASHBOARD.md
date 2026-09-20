# Родной веб-дашборд Hermes: сборка и установка

Пакет: `dashboard/hermes-dashboard-web-2026.5.7.zip` (5 МБ, 139 файлов).
Собран из официальных исходников `github.com/NousResearch/hermes-agent`, тег **v2026.5.7**
(коммит `498bfc7`) — ровно та версия, что стоит в контейнере. Ничего не правилось.

SHA-256: `0851e48acd4f073e6776377505ae97f9479b044ab048686cf47efac72188618f`

Внутри:
- `hermes_cli/web_dist/` — сам дашборд (страницы Чат, Сессии, Модели, Конфиг, .env, Cron, Логи, Аналитика, Навыки, Плагины, Профили, Документация);
- `ui-tui/dist/` — вкладка чата в браузере (флаг `--tui` в юните).

Зачем: банковский бандл содержит только исходники `web/` и `ui-tui/`, а собранных папок нет —
поэтому дашборд с `--skip-build` не запускался. Сборка требует npm и интернета, отсюда офлайн-пакет.

Метка: ⏳ — в контейнере ещё не запускалось.

## 1. Версии кода: ссылка вместо папки ⏳

Чтобы обновление и откат были переключением ссылки, а не переустановкой.

```bash
sudo systemctl stop 'hermes-dashboard@*' 2>/dev/null                 # остановить сервисы, если запущены
sudo mv /opt/hermes /opt/hermes-2026.5.7                             # текущая установка получает имя версии
sudo ln -sfn /opt/hermes-2026.5.7 /opt/hermes                        # /opt/hermes — ссылка на активную версию (-n: не залезать внутрь старой ссылки)
ls -l /opt/ | grep hermes                                            # ждём: hermes -> /opt/hermes-2026.5.7
sudo -u nobody env HOME=/tmp hermes-run --version                    # пути не изменились: ждём v0.13.0
```

Обновление в будущем: распаковать новую версию в `/opt/hermes-<версия>`, собрать для неё веб,
переключить ссылку `sudo ln -sfn /opt/hermes-<версия> /opt/hermes`, перезапустить сервисы.
Откат — вернуть ссылку на прежнюю папку.

## 2. Положить сборку ⏳

```bash
sha256sum ~/hermes-dashboard-web-2026.5.7.zip     # сверить с хешем выше
cd /opt/hermes-2026.5.7/hermes-agent              # в папку установленной версии (не через ссылку — так нагляднее)
sudo /opt/hermes/hermes-agent/venv/bin/python -m zipfile -e ~/hermes-dashboard-web-2026.5.7.zip .   # распаковать: web_dist и ui-tui/dist встанут на свои места
sudo chown -R root:root hermes_cli/web_dist ui-tui/dist
sudo chmod -R go-w,a+rX hermes_cli/web_dist ui-tui/dist
ls hermes_cli/web_dist/index.html ui-tui/dist/entry.js               # ждём оба файла
```

## 2а. Библиотеки для вкладки чата ⏳

Симптом без этого шага: дашборд работает, но вкладка Chat показывает `Chat unavailable: 1`.

Причина: вкладка чата запускает `node ui-tui/dist/entry.js`, которому нужны рабочие библиотеки
(`ui-tui/node_modules`, в том числе `@hermes/ink`). Не найдя их, Hermes пытается выполнить
`npm install` в `/opt/hermes` — без интернета и без прав на запись это завершается кодом 1.

Пакет: `dashboard/hermes-tui-node-modules-2026.5.7.zip` (8 МБ) — только runtime-зависимости
(`npm ci --omit=dev`), без инструментов сборки.
SHA-256: `ac7c332801a1b7a8cf41c59bd2a02538bc6e40a4c2d519ef6c73b86a5edb96fb`

```bash
sha256sum ~/hermes-tui-node-modules-2026.5.7.zip        # сверить с хешем выше
cd /opt/hermes-2026.5.7/hermes-agent
sudo /opt/hermes/hermes-agent/venv/bin/python -m zipfile -e ~/hermes-tui-node-modules-2026.5.7.zip .   # встанет в ui-tui/node_modules
# Hermes сверяет package-lock.json с составом node_modules и при расхождении запускает npm install.
# В поставке только runtime-зависимости, поэтому убираем файл сверки: при его отсутствии проверка
# считает, что устанавливать нечего (hermes_cli/main.py, _tui_need_npm_install).
sudo mv ui-tui/package-lock.json ui-tui/package-lock.json.offline
sudo chown -R root:root ui-tui/node_modules && sudo chmod -R go-w,a+rX ui-tui/node_modules
ls ui-tui/node_modules/@hermes/ink/package.json          # ждём: файл есть
sudo systemctl restart hermes-dashboard@agent            # перезапустить дашборд
```

Проверка: вкладка Chat открывается и отвечает.

## 3. Запустить для себя ⏳

```bash
sudo install -d -m 755 /etc/hermes/users
printf 'PORT=9121\nENABLED=yes\n' | sudo tee /etc/hermes/users/agent.conf >/dev/null
sudo systemctl start hermes-dashboard@agent
sudo systemctl status hermes-dashboard@agent --no-pager | head -12
sudo journalctl -u hermes-dashboard@agent -n 40 --no-pager     # если не active — причина здесь
sudo ss -tlnp | grep 9121                                      # ждём 127.0.0.1:9121
```

## 4. Открыть с VDI ⏳

```
ssh -p 215 -N -L 9121:127.0.0.1:9121 <учётка>@<IP>
```
Браузер: `http://localhost:9121`. Токен, если попросит, — в `/home/hermes-agent/.hermes/.env`.

## Известные тонкости

- Дашборд запускается с `--skip-build`: без собранного `web_dist` он падал.
- Вкладка чата работает через встроенный терминал (PTY): нужны и сборка `ui-tui/dist`, и её
  runtime-зависимости `ui-tui/node_modules` (шаг 2а).
- Страницы «Проекты» в дашборде нет ни в этой, ни в свежей версии — это функция стороннего WebUI.
