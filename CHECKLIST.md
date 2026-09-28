# Что делать в банке после одобрения ИБ

Пошагово, по порядку. Отмечай галочками по мере выполнения.
Подробности каждого шага — в `UPGRADE.md`; здесь только последовательность и команды.

Все команды выполняются со своей учётной записи через `sudo`.
**Никогда не запускать Hermes под своей учёткой** — у неё sudo, агент получил бы root.

---

## 0. Перед началом (10 минут)

- [ ] Архив занесён в контейнер по SFTP, лежит в домашнем каталоге
- [ ] Сверить контрольную сумму — **файл должен совпасть до символа**:

```bash
sha256sum ~/hermes-2026.8.31-offline.zip
# ждём: c92ca56c1a513fe2bb115fa74882a5ddeec4f29e877a465f4ac286a01b102a98
```

- [ ] Предупредить пользователей: «сегодня будет обновление, перерыв несколько минут»

---

## 1. Распаковать рядом со старой версией (5 минут)

Ничего не перезаписываем: новая версия встаёт в отдельный каталог.

```bash
sudo install -d -m 755 /opt/hermes-2026.8.31
sudo /opt/hermes/hermes-agent/venv/bin/python -m zipfile -e \
     ~/hermes-2026.8.31-offline.zip /tmp/bundle/          # распаковать во временный каталог
sudo cp -r /tmp/bundle/hermes-2026.8.31/. /opt/hermes-2026.8.31/
sudo chown -R root:root /opt/hermes-2026.8.31
sudo chmod -R go-w,a+rX /opt/hermes-2026.8.31             # код только на чтение
```

- [ ] Каталог `/opt/hermes-2026.8.31` создан, внутри есть `hermes_cli/web_dist/index.html`

---

## 2. Создать окружение и поставить библиотеки (10 минут)

Окружение строится **Python 3.11** — под него собраны библиотеки. Он уже есть в старой установке.

```bash
sudo /opt/hermes-2026.5.7/tools/python/bin/python3 -m venv /opt/hermes-2026.8.31/venv

sudo /opt/hermes-2026.8.31/venv/bin/pip install --no-index --find-links /tmp/bundle/wheels \
  openai "httpx[socks]" pydantic certifi python-dotenv fire rich tenacity pyyaml ruamel.yaml \
  requests jinja2 firecrawl-anydoc prompt_toolkit croniter snowballstemmer packaging Markdown \
  "PyJWT[crypto]" urllib3 cryptography psutil websockets pathspec Pillow fastapi \
  "uvicorn[standard]" python-multipart ptyprocess

sudo /opt/hermes-2026.8.31/venv/bin/pip install --no-index --find-links /tmp/bundle/wheels-office \
  openpyxl XlsxWriter python-docx python-pptx

cd /opt/hermes-2026.8.31 && sudo ./venv/bin/python -m hermes_cli.main --version
```

- [ ] Печатает `Hermes Agent v0.21.0 (2026.8.31)`

> Если пишет «No module named 'hermes_cli'» — забыли `cd`. Hermes запускается **из каталога установки**.

---

## 3. Обновить обёртку запуска (2 минуты)

```bash
sudo cp -a /usr/local/bin/hermes-run /usr/local/bin/hermes-run.bak-$(date +%F)
sudo tee /usr/local/bin/hermes-run >/dev/null <<'EOF'
#!/bin/sh
# Hermes запускается из каталога установки: пакет не ставится в окружение.
. /etc/hermes/hermes.env 2>/dev/null
cd /opt/hermes || exit 1
exec /opt/hermes/venv/bin/python -m hermes_cli.main "$@"
EOF
sudo chmod 755 /usr/local/bin/hermes-run
```

- [ ] Файл обновлён, старый сохранён с датой

---

## 4. Переключиться на новую версию (2 минуты)

```bash
sudo systemctl stop 'hermes-dashboard@*'                  # остановить всех
sudo ln -sfn /opt/hermes-2026.8.31 /opt/hermes            # переключить ссылку
ls -l /opt/ | grep 'hermes ->'                            # ждём: hermes -> /opt/hermes-2026.8.31
sudo systemctl start hermes-dashboard@agent               # поднять только себя
```

- [ ] Ссылка указывает на новую версию
- [ ] Свой экземпляр запустился

**Если что-то пошло не так — откат за десять секунд:**

```bash
sudo systemctl stop 'hermes-dashboard@*'
sudo ln -sfn /opt/hermes-2026.5.7 /opt/hermes
sudo systemctl start 'hermes-dashboard@*'
```

---

## 5. Проверить на себе, до того как пускать людей (10 минут)

```bash
hermes-run --version                                       # новая версия
sudo -iu hermes-agent hermes-run -z "Ответь одним словом: работает"   # модель отвечает
sudo -iu hermes-agent hermes-run logs errors --since 10m   # ошибок нет
sudo ss -tlnp | grep 912                                   # порт слушается на 127.0.0.1
```

- [ ] Модель ответила (это главное: сервис доступен, ключ верный, сертификат принят)
- [ ] Веб открывается по ссылке, вход работает
- [ ] Вкладка Chat отвечает
- [ ] Страница **Files** открывается, файл скачивается
- [ ] В юнитах нет флага `--insecure`:

```bash
sudo grep -r 'insecure' /etc/systemd/system/hermes-*.service && echo 'НАЙДЕН — убрать' || echo 'чисто'
```

---

## 6. Поднять остальных (2 минуты)

```bash
sudo systemctl start 'hermes-dashboard@*'
sudo hermes-user list                                      # все active
```

- [ ] Все экземпляры работают
- [ ] Один из коллег подтвердил, что зашёл и чат отвечает

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
sudo systemctl daemon-reload && sudo systemctl restart 'hermes-dashboard@*'
```

### 7.3. Задачи по расписанию (кроны)

Без шлюза задачи создаются, но не срабатывают.

```bash
sudo install -m 644 ~/deploy/hermes-gateway@.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now hermes-gateway@agent
sudo -iu hermes-agent hermes-run cron status   # ждём: Gateway is running
```

**Важно:** при создании задачи указывать рабочий каталог явно, иначе агент окажется
в каталоге программы и не сможет писать:

```bash
sudo -iu hermes-agent hermes-run cron create "0 9 * * 1" "Собери сводку за неделю" \
     --name "недельная сводка" --workdir /srv/hermes/agent
```

### 7.4. Прочее

- [ ] Сменить веб-пароль учётки `agent` (его знает администратор с отладки)
- [ ] Убрать тестовых пользователей, если остались: `sudo hermes-user purge testuser`
- [ ] Обновить `deploy/` в контейнере: скопировать свежие документы по SFTP

---

## 8. Если что-то сломалось

| Симптом | Что смотреть |
|---|---|
| `No module named 'hermes_cli'` | забыт `cd /opt/hermes` — использовать `hermes-run` |
| `Connection error` при рабочем `curl` | четыре строки `SSL_*`/`*_CA_BUNDLE` в `.env` пользователя |
| Настройки «не применяются» | `hermes-run logs errors` → сломан YAML, копия сохранена рядом |
| Экземпляр не стартует | `sudo journalctl -u hermes-dashboard@<имя> -n 50` |
| Веб открылся, чат молчит | заголовки в nginx: `X-Forwarded-*` должны быть пустыми |
| Задачи по расписанию не срабатывают | не запущен `hermes-gateway@<имя>` |
| Ничего не помогает | откат из шага 4, затем разбираться спокойно |

---

## Сколько это займёт

Шаги 0–6 — около 40 минут, из них перерыв для пользователей 5–10 минут.
Шаг 7 можно делать позже, он никого не останавливает.
