# Что делать в банке после одобрения ИБ

Пошагово, по порядку. Отмечается галочками по мере выполнения.
Обоснование схемы — в `UPGRADE.md`; здесь только последовательность и команды.

Все команды выполняются со своей учётной записи через `sudo`.
**Никогда не запускать Hermes под своей учёткой** — у неё sudo, агент получил бы root.
Hermes проверяем только от имени служебных учёток: `sudo -iu hermes-probnik ...`.

> **Длинные команды лучше не набирать в терминале руками.** Многострочные блоки при вставке
> в SSH-сессию часто перерисовываются криво и обрываются на середине. Надёжнее: сохранить
> команду в файл на своём компьютере, занести по SFTP и выполнить `bash имя-файла.sh`.

## Схема в двух словах

Состояние контейнера на 06.10.2026: `/opt/hermes -> /opt/hermes-2026.5.7`, работают только дашборды,
шлюзов и webui нет, node v24.15.0.

- Новая версия ставится **рядом**, в `/opt/hermes-2026.8.31`. Старая не трогается.
- Окружение новой версии лежит **по тому же относительному пути**, что у старой:
  `hermes-agent/venv`. Поэтому `HERMES_PYTHON` в `hermes.env` и обёртку `hermes-run` менять не нужно.
- В новом комплекте **нет** node, rg, ffmpeg, Python и браузеров — они берутся из старой версии
  по абсолютным путям (шаг 1). Каталог `tools/` в новой версии — это Python-код Hermes, не программы.
- **Переключение и откат — одна команда:** перевесить ссылку `/opt/hermes`.

> ⛔ **`/opt/hermes-2026.5.7` не удалять никогда, пока не внесены свои node и Python.**
> Новая версия берёт оттуда Python 3.11, node, rg, ffmpeg и браузеры.

---

## 0. Перед началом (10 минут, пользователи не затронуты)

- [ ] Архив занесён в контейнер по SFTP, лежит в домашнем каталоге
- [ ] Сверить контрольную сумму — **должна совпасть до символа**:

```bash
sha256sum ~/hermes-2026.8.31-offline.zip
# ждём: 5a50ecbf6d70a1306b4dd5d542753a78ca07c85e3cb8a44ab6716208bbd27133
```

- [ ] Сохранить копии всего, что будем менять (на случай отката):

```bash
sudo install -d -m 700 /root/hermes-upgrade                         # папка для копий, видна только root
sudo cp -a /etc/hermes/hermes.env /root/hermes-upgrade/             # настройки обёртки — правим в шаге 1
sudo cp -a /usr/local/bin/hermes-run /root/hermes-upgrade/          # обёртку не меняем, копия на всякий случай
sudo systemctl list-units 'hermes-dashboard@*' --state=running --plain --no-legend \
  | awk '{print $1}' | sudo tee /root/hermes-upgrade/running.txt    # список работающих экземпляров — по нему поднимаем
sudo cat /root/hermes-upgrade/running.txt                           # все, кто сейчас работает; запомнить число строк
```

---

## 1. Пути к программам — абсолютные (5 минут, пользователи не затронуты)

Сейчас `hermes.env` ссылается на `/opt/hermes/tools/...`. После переключения ссылки там окажется
код новой версии, и node с браузерами пропадут. Прописываем пути к старой версии напрямую —
**сегодня это те же самые файлы**, поэтому для работающих людей ничего не меняется.

```bash
# sed меняет текст в файле; -i.orig оставляет рядом копию hermes.env.orig
sudo sed -i.orig \
  -e 's#/opt/hermes/tools#/opt/hermes-2026.5.7/tools#g' \
  -e 's#/opt/hermes/ms-playwright#/opt/hermes-2026.5.7/ms-playwright#' \
  /etc/hermes/hermes.env
cat /etc/hermes/hermes.env
```

Ждём ровно это:

```
HERMES_PYTHON=/opt/hermes/hermes-agent/venv/bin/python
HERMES_PATH_PREPEND=/opt/hermes-2026.5.7/tools/node/bin:/opt/hermes-2026.5.7/tools
PLAYWRIGHT_BROWSERS_PATH=/opt/hermes-2026.5.7/ms-playwright
HERMES_GIT_BASH_PATH=/bin/bash
```

Проверка на тестовой учётке — работающие дашборды файл перечитают только при перезапуске:

```bash
sudo systemctl restart hermes-dashboard@probnik             # перечитает hermes.env
sudo systemctl is-active hermes-dashboard@probnik                # ждём active
sudo -iu hermes-probnik hermes-run --version                # ждём v0.13.0 — старая версия на новых путях
```

- [ ] `hermes.env` как выше, `probnik` поднялся

---

## 2. Распаковать рядом со старой версией (5 минут, пользователи не затронуты)

```bash
sudo rm -rf /tmp/bundle                                    # на случай остатков прошлой попытки
sudo /opt/hermes/hermes-agent/venv/bin/python -m zipfile -e \
     ~/hermes-2026.8.31-offline.zip /tmp/bundle/            # распаковать во временный каталог
ls /tmp/bundle                                             # ждём: README.md SHA256SUMS.txt hermes-2026.8.31 wheels wheels-office
cd /tmp/bundle && sha256sum -c --quiet SHA256SUMS.txt && echo 'файлы целы'; cd ~

sudo install -d -m 755 /opt/hermes-2026.8.31
sudo cp -r /tmp/bundle/hermes-2026.8.31/. /opt/hermes-2026.8.31/
ls /opt/hermes-2026.8.31/hermes_cli/web_dist/index.html /opt/hermes-2026.8.31/hermes_cli/tui_dist/entry.js
```

- [ ] Обе проверки прошли, оба файла на месте

---

## 3. Окружение и библиотеки (10 минут, пользователи не затронуты)

Окружение строится на **Python 3.11 из старой версии** — под него собраны библиотеки.

```bash
/opt/hermes-2026.5.7/tools/python/bin/python3.11 --version   # ждём Python 3.11.x

# То же место, что у старой версии: hermes-agent/venv — тогда hermes.env менять не нужно
sudo /opt/hermes-2026.5.7/tools/python/bin/python3.11 -m venv /opt/hermes-2026.8.31/hermes-agent/venv
V=/opt/hermes-2026.8.31/hermes-agent/venv

sudo $V/bin/pip install --no-index --find-links /tmp/bundle/wheels \
  openai "httpx[socks]" pydantic certifi python-dotenv fire rich tenacity pyyaml ruamel.yaml \
  requests jinja2 firecrawl-anydoc prompt_toolkit croniter snowballstemmer packaging Markdown \
  "PyJWT[crypto]" urllib3 cryptography psutil websockets pathspec Pillow fastapi \
  "uvicorn[standard]" python-multipart ptyprocess

sudo $V/bin/pip install --no-index --find-links /tmp/bundle/wheels-office \
  openpyxl XlsxWriter python-docx python-pptx

# Пакет Hermes не ставится в окружение. Файл .pth — стандартный механизм Python: путь из него
# добавляется к поиску модулей, и код виден из любого каталога. Без этого нужен был бы cd
# в каталог программы, и агент работал бы там, а не в своей папке /srv/hermes/<имя>.
echo /opt/hermes-2026.8.31 | sudo tee $V/lib/python3.11/site-packages/hermes-code.pth

sudo chown -R root:root /opt/hermes-2026.8.31
sudo chmod -R go-w,a+rX /opt/hermes-2026.8.31             # код только на чтение
sudo rm -rf /tmp/bundle                                   # распакованный архив больше не нужен
```

Проверка — от тестовой учётки, из её домашнего каталога (не из каталога программы):

```bash
sudo -iu hermes-probnik /opt/hermes-2026.8.31/hermes-agent/venv/bin/python -m hermes_cli.main --version
```

- [ ] Печатает `Hermes Agent v0.21.0 (2026.8.31)`

> «No module named 'hermes_cli'» — не создан или не там лежит `hermes-code.pth`.
> Проверить: `ls /opt/hermes-2026.8.31/hermes-agent/venv/lib/` — должна быть `python3.11`.

**До этого места пользователи ничего не заметили.** Можно остановиться и продолжить в другой день.

---

## 4. Переключение (5–10 минут перерыва для пользователей)

- [ ] Предупредить пользователей: «перерыв несколько минут»

```bash
sudo systemctl stop 'hermes-dashboard@*'                   # остановить всех
sudo systemctl list-units 'hermes-dashboard@*' --state=running --no-legend | wc -l   # ждём 0

# Копия данных пользователей: новая версия при первом запуске может перестроить базу
# и конфиги, и старая на них уже не стартует. Делается при остановленных сервисах.
sudo bash -c 'tar czf /root/hermes-upgrade/home-hermes-$(date +%F).tar.gz /home/hermes-*/.hermes'   # glob раскрывает root: чужие хомяки закрыты
sudo ls -lh /root/hermes-upgrade/                          # архив есть и не пустой

sudo ln -sfn /opt/hermes-2026.8.31 /opt/hermes             # переключить ссылку
readlink /opt/hermes                                       # ждём /opt/hermes-2026.8.31
sudo systemctl start hermes-dashboard@probnik              # поднять только тестовую учётку
```

---

## 5. Проверить на тестовой учётке, до того как пускать людей (10 минут)

```bash
sudo systemctl is-active hermes-dashboard@probnik                           # ждём active
sudo -iu hermes-probnik hermes-run --version                           # ждём v0.21.0
sudo -iu hermes-probnik hermes-run -z "Ответь одним словом: работает"  # модель отвечает
sudo -iu hermes-probnik hermes-run logs errors --since 10m             # ошибок нет
sudo journalctl -u hermes-dashboard@probnik -n 30 --no-pager           # нет 'No module', 'node', 'Traceback'
```

- [ ] Модель ответила (это главное: сервис доступен, ключ верный, сертификат принят)
- [ ] Веб `probnik` открывается по ссылке, вход работает
- [ ] Вкладка Chat отвечает (это проверка node)
- [ ] Страница **Files** открывается, файл скачивается
- [ ] В юнитах нет флага `--insecure`:

```bash
sudo grep -r 'insecure' /etc/systemd/system/hermes-*.service && echo 'НАЙДЕН — убрать' || echo 'чисто'
```

**Любой пункт не прошёл — откат (ниже), разбираться потом спокойно.**

---

## 6. Поднять остальных (2 минуты)

```bash
sudo xargs -a /root/hermes-upgrade/running.txt systemctl start   # ровно те, кто работал до обновления
sudo systemctl list-units 'hermes-dashboard@*' --state=running --no-legend | wc -l   # ждём столько же, сколько строк в running.txt
```

- [ ] Работают все экземпляры из `running.txt`
- [ ] Один из коллег подтвердил, что зашёл и чат отвечает

---

## Откат (одна минута)

Обёртка и `hermes.env` после шага 1 подходят обеим версиям, поэтому откат — только ссылка:

```bash
sudo systemctl stop 'hermes-dashboard@*'
sudo ln -sfn /opt/hermes-2026.5.7 /opt/hermes
sudo xargs -a /root/hermes-upgrade/running.txt systemctl start
sudo systemctl list-units 'hermes-dashboard@*' --state=running --no-legend | wc -l   # ждём столько же, сколько строк в running.txt
```

**Если старая версия после отката не стартует** (в журнале ошибки базы или конфига) — новая
успела перестроить данные. Вернуть их из копии, затем снова поднять:

```bash
sudo systemctl stop 'hermes-dashboard@*'
sudo ls /root/hermes-upgrade/                                       # узнать дату в имени архива
sudo tar xzf /root/hermes-upgrade/home-hermes-ДАТА.tar.gz -C /     # tar хранит пути без ведущего /
sudo xargs -a /root/hermes-upgrade/running.txt systemctl start
```

Возврат данных стирает всё, что пользователи успели сделать на новой версии, — только если
без этого старая не работает.

---

## 7. Донастройка (можно на следующий день)

### 7.1. Часовой пояс в профили-роли

Конфигурация профиля создаётся без пояса, и роль считает время по UTC.

```bash
sudo bash -c 'for u in /home/hermes-*; do
  for p in "$u"/.hermes/profiles/*/config.yaml; do
    [ -e "$p" ] || continue
    grep -q "^timezone:" "$p" || echo "timezone: Europe/Moscow" >> "$p"
  done
done; echo готово'
```

### 7.2. Запереть файловый раздел в рабочей папке

Второй рубеж: основной — права операционной системы.

```bash
sudo systemctl edit hermes-dashboard@          # добавить в [Service]:
# Environment=HERMES_DASHBOARD_FILES_ROOT=/srv/hermes/%i
sudo systemd-analyze verify hermes-dashboard@probnik.service   # ждём пустой вывод
sudo systemctl daemon-reload && sudo xargs -a /root/hermes-upgrade/running.txt systemctl restart
```

### 7.3. Задачи по расписанию (кроны)

Без шлюза задачи создаются, но не срабатывают. Юнит шлюза запускает Hermes через ту же
обёртку `hermes-run`, что и дашборд.

```bash
sudo install -m 644 ~/deploy/hermes-gateway@.service /etc/systemd/system/
sudo systemd-analyze verify hermes-gateway@agent.service   # ОБЯЗАТЕЛЬНО: ждём пустой вывод
sudo systemctl daemon-reload
sudo systemctl enable --now hermes-gateway@agent
sudo -iu hermes-agent hermes-run cron status   # ждём: Gateway is running
```

> **Почему проверка обязательна.** systemd не понимает комментарии в конце строки: всё после
> значения попадает в само значение. Юнит при этом стартует и выглядит рабочим, а песочница
> молча отключена. Пустой вывод `systemd-analyze verify` — единственное доказательство,
> что ограничения применились. Делать после любой правки любого юнита.

Задача начинается в назначенную минуту или в течение следующей — планировщик проверяет
расписание раз в минуту. Контроль отказов — `cron runs` и `cron incidents`: `cron status`
светит зелёным даже после неудачных запусков.

**Важно:** при создании задачи указывать рабочий каталог явно:

```bash
sudo -iu hermes-agent hermes-run cron create "0 9 * * 1" "Собери сводку за неделю" \
     --name "недельная сводка" --workdir /srv/hermes/agent

# Проверка часового пояса: в ответе должно быть +03:00, а не +00:00
sudo -iu hermes-agent hermes-run cron create "0 9 * * *" "проверка" --name tz-check
sudo -iu hermes-agent hermes-run cron list
```

Если время показано как `+00:00` — в конфигурации не хватает `timezone`, и расписание
съедет на три часа.

### 7.4. Прочее

- [ ] Убрать тестовых пользователей, если остались: `sudo hermes-user purge testuser`
- [ ] Обновить `deploy/` в контейнере: скопировать свежие документы по SFTP
- [ ] Через неделю спокойной работы: удалить копию данных `/root/hermes-upgrade/home-hermes-*.tar.gz`.
      **Каталог `/opt/hermes-2026.5.7` не удалять** — см. начало документа.

---

## 8. Если что-то сломалось

| Симптом | Что смотреть |
|---|---|
| `No module named 'hermes_cli'` | нет `hermes-code.pth` в `hermes-agent/venv/lib/python3.11/site-packages/` |
| Чат молчит, в журнале `node` / `ENOENT` | `hermes.env`: пути к `tools` должны вести в `/opt/hermes-2026.5.7` |
| `Connection error` при рабочем `curl` | четыре строки `SSL_*`/`*_CA_BUNDLE` в `.env` пользователя |
| Настройки «не применяются» | `hermes-run logs errors` → сломан YAML, копия сохранена рядом |
| Экземпляр не стартует | `sudo journalctl -u hermes-dashboard@<имя> -n 50` |
| Веб открылся, чат молчит | заголовки в nginx: `X-Forwarded-*` должны быть пустыми |
| Задачи по расписанию не срабатывают | не запущен `hermes-gateway@<имя>` |
| Ничего не помогает | откат, затем разбираться спокойно |

---

## Сколько это займёт

Шаги 0–3 — около 30 минут, пользователи ничего не замечают; их можно сделать заранее.
Шаги 4–6 — около 20 минут, из них перерыв для пользователей 5–10 минут.
Шаг 7 можно делать позже, он никого не останавливает.
