# Hermes WebUI в контейнере

Пакет: `webui/hermes-webui-0.51.69-offline.zip` — WebUI v0.51.69 (15.05.2026) под Hermes v0.13.0,
все CDN-библиотеки внутри, интернет не нужен. Что изменено против оригинала — `OFFLINE-PATCH.md` в архиве.
SHA-256: `b9a544b4b76dfec790f6616f2fe870df01e6093380aa03bcea6ed5c99965121c`

Метка: ⏳ — ещё не запускалось в контейнере.

## 1. Занести и распаковать ⏳

WinSCP: `hermes-webui-0.51.69-offline.zip` и `deploy/hermes-webui@.service` → в свой хомяк.

```bash
sha256sum ~/hermes-webui-0.51.69-offline.zip           # сверить с хешем выше — файл дошёл целым
sudo /opt/hermes/hermes-agent/venv/bin/python -m zipfile -e ~/hermes-webui-0.51.69-offline.zip /opt/   # распаковать (unzip может не быть)
sudo mv /opt/hermes-webui-0.51.69 /opt/hermes-webui
sudo chown -R root:root /opt/hermes-webui && sudo chmod -R go-w,a+rX /opt/hermes-webui   # код только на чтение
```

## 2. Юнит ⏳

```bash
sudo install -m 644 ~/hermes-webui@.service /etc/systemd/system/
sudo install -d -m 755 /etc/hermes/users
printf 'PORT=9121\nENABLED=yes\n' | sudo tee /etc/hermes/users/agent.conf >/dev/null   # порт для hermes-agent
sudo systemctl daemon-reload
sudo systemctl start hermes-webui@agent
sudo systemctl status hermes-webui@agent --no-pager | head -15
sudo journalctl -u hermes-webui@agent -n 40 --no-pager                                 # если не active — причина здесь
sudo ss -tlnp | grep 9121                                                               # ждём 127.0.0.1:9121
```

## 3. Посмотреть с VDI ⏳

```
ssh -p 215 -N -L 9121:127.0.0.1:9121 <учётка>@<IP контейнера>
```
Браузер: `http://localhost:9121`. Проверить: чат отвечает, код подсвечивается (значит vendor-библиотеки грузятся).

## Дальше

- Когда заработает — `hermes-user` переключается с `hermes-dashboard@` на `hermes-webui@`.
- Вход для коллег — nginx по `deploy/nginx-hermes.conf` (логин → порт).
