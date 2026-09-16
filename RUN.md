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

Посмотреть дашборд без nginx — туннель с VDI (окно не закрывать):
```
ssh -p 215 -N -L 9121:127.0.0.1:9121 <учётка>@<IP контейнера>
```
и в браузере `http://localhost:9121`.

## Пользователи

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

## Если что-то не так

| Симптом | Где смотреть / что делать |
|---|---|
| `Unknown provider` | в `config.yaml` должно быть `model.provider: custom` |
| `Connection error` | в `.env` агента четыре строки `SSL_*` / `*_CA_BUNDLE`; `logs errors` |
| Настройки не действуют | `logs errors` → «Falling back to default config» = сломан YAML |
| Hermes читает не тот конфиг | запускать с `sudo -iu` (буква `i`); `config show` → раздел Paths |
| `~` указывает не туда | в `sudo -iu X команда ~/...` тильду разворачивает **твой** shell — писать полный путь |
| После правки YAML всё сломалось | перезаписать файл целиком (`sudo tee`), не удалять строки по номерам |

Правило поиска ошибок: сначала `logs errors`, потом `doctor`, потом уже гипотезы.
