# hermes-deploy

Многопользовательская установка Hermes Agent в LXC-контейнере (Debian 13):
у каждого пользователя своя Linux-учётка без sudo, своя память и свой дашборд
в systemd-песочнице; вход через nginx по логину.

| Файл | Куда | Что делает |
|---|---|---|
| `hermes-user` | `/usr/local/sbin/` | Завести / отключить / удалить пользователя одной командой |
| `hermes-dashboard@.service` | `/etc/systemd/system/` | Клетка: дашборд пользователя с лимитами и изоляцией |
| `hermes.slice` | `/etc/systemd/system/` | Общий потолок ресурсов на всех агентов |
| `hermes-run` | `/usr/local/bin/` | Запуск общего кода Hermes со своим `HERMES_HOME` |
| `config.template.yaml` | `/etc/hermes/` | Безопасный `config.yaml` для новых пользователей |
| `nginx-hermes.conf` | `/etc/nginx/conf.d/` | basic-auth за корпоративным прокси, логин → свой порт (копия INSTALL.md, 10.3) |

Секретов в репозитории нет: ключи модели живут в `/etc/hermes/model.env` на сервере.
