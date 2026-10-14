# Proxy

## Шаг 1. Установите cntlm

```bash
sudo dnf install cntlm
```

---

## Шаг 2. Настройте cntlm

Отредактируйте конфиг:

```bash
sudo nano /etc/cntlm.conf
```

Заполните примерно так (подставьте свои данные):

```text
Username        ваш_логин_AD
Domain          ваш_домен_AD      # например, COMPANY или COMPANY.LOCAL
Passwd          ваш_пароль_AD     # вводите один раз; cntlm сам посчитает хэши
Proxy           proxy.company.ru:3128
Listen          3129              # локальный порт, который будет слушать cntlm
Auth            NTLMv2
Flags           0x07820000
```

> ⚠️ Пароль в `cntlm.conf` нужен только для генерации NTLM‑хэшей. После настройки лучше убрать строку `Passwd` из конфига и хранить учётные данные иначе (или использовать отдельный скрипт запуска), но для старта можно оставить.

Сохраните, затем запустите и включите:

```bash
sudo systemctl enable --now cntlm
```

Проверьте, что порт занят:

```bash
ss -tlnp | grep 3129
```

---

## Шаг 3. Настройте CLI‑утилиты на localhost:3129

В `~/.bashrc`:

```bash
export http_proxy="http://127.0.0.1:3129"
export https_proxy="http://127.0.0.1:3129"
export ftp_proxy="http://127.0.0.1:3129"
export no_proxy="localhost,127.0.0.1,::1,.company.ru"
```

Примените:

```bash
source ~/.bashrc
```

Проверка:

```bash
curl -I http://example.com
```

Если видите HTTP‑ответ — cntlm успешно делает NTLM‑аутентификацию.

---

## Шаг 4. Настройте dnf на тот же локальный прокси

```bash
sudo mkdir -p /etc/dnf/dnf.conf.d
sudo nano /etc/dnf/dnf.conf.d/proxy.conf
```

Содержимое:

```text
[main]
proxy=http://127.0.0.1:3129
# proxy_username и proxy_password НЕ нужны
```

Проверка:

```bash
sudo dnf makecache
sudo dnf update
```

---

## Шаг 5. sudo и переменные окружения

Чтобы `sudo curl` тоже шёл через прокси, добавьте в `sudoers`:

```bash
sudo visudo
```

Добавьте в конец:

```text
Defaults env_keep += "http_proxy https_proxy ftp_proxy no_proxy"
```

Теперь `sudo curl -I http://example.com` должен работать.

---

## Важные нюансы именно для RedOS

- **SELinux.** Если после настройки `dnf` или `curl` выдают странные ошибки доступа — проверьте:
  ```bash
  journalctl -xe
  ausearch -m avc
  ```
  Иногда SELinux блокирует сетевые действия прокси. Для проверки можно временно перевести SELinux в permissive: `sudo setenforce 0` (не для продакшена).
- **DNS.** Убедитесь, что `proxy.company.ru` резолвится:
  ```bash
  nslookup proxy.company.ru
  ```
- **Файрвол.** Проверьте, что исходящие соединения на порт 3128 разрешены (это нужно для связи `cntlm` с корпоративным прокси):
  ```bash
  sudo firewall-cmd --list-all
  ```

---

## Как понять, что всё работает

1. **CLI:** `curl -I http://example.com` → HTTP‑ответ.
2. **sudo + прокси:** `sudo curl -I http://example.com` → HTTP‑ответ.
3. **dnf:** `sudo dnf makecache` → успешное скачивание метаданных.
4. **Локальные ресурсы:** `curl http://internal.company.ru` → не должен идти через прокси (если домен в `no_proxy`).

---
