# NUT config for Proxmox homelab

Конфигурация [Network UPS Tools](https://networkupstools.org/) для домашней инфраструктуры с ИБП CyberPower UT2200EG (USB HID, USB ID `0764:0501`), подключённым по USB к основному PVE-хосту. Сетап трёхуровневый: один primary с физически подключённым ИБП раздаёт статус по сети, secondary-хосты (второй Proxmox и PBS) получают команду на выключение через сеть. При длительном отключении питания все хосты корректно и синхронно завершают работу.

## Топология

| Хост | Роль | IP | Что делает |
|---|---|---|---|
| PVE-Main | primary | 192.168.10.12 | ИБП подключён по USB, драйвер + upsd + upsmon, командует shutdown |
| PVE-Mini | secondary | 192.168.10.11 | upsmon-netclient, гасится по FSD от primary |
| PBS | secondary | 192.168.10.15 | upsmon-netclient, гасится по FSD от primary |

Дашборд PeaNUT (опционально) читает статус ИБП с primary как ещё один сетевой клиент.

## Что внутри

| Файл | Назначение |
|---|---|
| `nut.conf` | Режим работы NUT (`netserver` на primary, `netclient` на secondary) |
| `ups.conf` | Описание UPS-устройства для драйвера (только primary) |
| `upsd.conf` | Настройки сетевого сервера `upsd` (только primary) |
| `upsd.users.example` | Шаблон пользователей с плейсхолдерами (только primary) |
| `upsmon.conf.example` | Шаблон монитора с плейсхолдером пароля |
| `upssched.conf` | Планировщик событий и таймеры (только primary) |
| `upssched-cmd` | Скрипт-обработчик для upssched (только primary) |
| `nut-driver@myups.service.d/fixperms.conf` | systemd drop-in для прав на USB-устройство (только primary) |

Реальные `upsd.users` и `upsmon.conf` в репо не хранятся — они в `.gitignore`, так как содержат пароли NUT-пользователей.

## На чём тестировалось

- Proxmox VE 9 и Proxmox Backup Server 4 (Debian 13 trixie)
- NUT 2.8.1
- ИБП CyberPower UT2200EG (2200 VA / 1320 W, Line-Interactive, AVR)
- Протокол связи: USB HID Power Device Class (драйвер `usbhid-ups`, субдрайвер CyberPower HID)

## Установка на primary (PVE-Main)

```bash
# 1. Установить NUT
apt install -y nut nut-client nut-server

# 2. Склонировать репо и разложить конфиги
git clone https://github.com/roman-kvasnikov/nut-config.git /tmp/nut-config
cd /tmp/nut-config

install -o root -g nut -m 640 nut.conf ups.conf upsd.conf upssched.conf /etc/nut/
install -o root -g nut -m 750 upssched-cmd /etc/nut/

# 3. Создать upsd.users и upsmon.conf из шаблонов
cp upsd.users.example /etc/nut/upsd.users
cp upsmon.conf.example /etc/nut/upsmon.conf
chown root:nut /etc/nut/upsd.users /etc/nut/upsmon.conf
chmod 640 /etc/nut/upsd.users /etc/nut/upsmon.conf

# Сгенерировать пароли и подставить в файлы
MONPASS=$(openssl rand -base64 18 | tr -d '/+=')
MINIPASS=$(openssl rand -base64 18 | tr -d '/+=')
PBSPASS=$(openssl rand -base64 18 | tr -d '/+=')
sed -i "s/CHANGE_ME_MONUSER_PASSWORD/$MONPASS/g" /etc/nut/upsd.users /etc/nut/upsmon.conf
sed -i "s/CHANGE_ME_MINI_PASSWORD/$MINIPASS/g" /etc/nut/upsd.users
sed -i "s/CHANGE_ME_PBS_PASSWORD/$PBSPASS/g" /etc/nut/upsd.users

# 4. Установить drop-in для прав на USB-устройство (см. секцию про udev ниже)
mkdir -p /etc/systemd/system/nut-driver@myups.service.d
install -m 644 nut-driver@myups.service.d/fixperms.conf /etc/systemd/system/nut-driver@myups.service.d/
systemctl daemon-reload

# 5. Включить и запустить
systemctl enable --now nut-driver.target nut-server.service nut-monitor.service

# 6. Проверить
upsc myups@localhost
```

## Установка на secondary (PVE-Mini, PBS)

На secondary-хостах нужен только клиент, без сервера и драйвера.

```bash
# 1. Установить только клиент
apt install -y nut-client

# 2. Режим netclient
echo "MODE=netclient" > /etc/nut/nut.conf

# 3. Настроить upsmon.conf (secondary указывает на primary)
cp upsmon.conf.secondary.example /etc/nut/upsmon.conf
chown root:nut /etc/nut/upsmon.conf
chmod 640 /etc/nut/upsmon.conf
# Подставить пароль пользователя этого хоста из upsd.users на primary
sed -i "s/CHANGE_ME_SECONDARY_PASSWORD/<PASSWORD>/g" /etc/nut/upsmon.conf

# 4. Включить и запустить
systemctl enable --now nut-monitor.service

# 5. Проверить чтение статуса с primary
upsc myups@192.168.10.12
```

Секция `MONITOR` в `upsmon.conf` на secondary указывает роль `secondary` и адрес primary:

```
MONITOR myups@192.168.10.12 1 <username> <PASSWORD> secondary
```

На primary в `upsd.users` для каждого secondary должен быть заведён пользователь с ролью `upsmon secondary`. Firewall primary (nftables) должен пропускать порт 3493 с IP-адресов secondary-хостов.

## ⚠️ Важно: права на USB-устройство (udev не срабатывает автоматически)

На свежей установке Proxmox системное udev-правило `62-nut-usbups.rules` из пакета `nut-server` содержит корректную запись для CyberPower (`GROUP="nut"`), но по факту **не применяется автоматически** — USB-устройство остаётся с правами `root:root`, и драйвер `usbhid-ups` (работающий под непривилегированным юзером `nut`) падает с:

```
libusb1: Could not open any HID devices: insufficient permissions on everything
```

**Лечение** — systemd drop-in `fixperms.conf`, который назначает группу `nut` на USB-устройство перед стартом драйвера:

```ini
[Service]
ExecStartPre=-/bin/sh -c 'DEV=$(lsusb | grep -i "0764:0501" | sed -E "s|Bus ([0-9]+) Device ([0-9]+):.*|/dev/bus/usb/\1/\2|"); [ -n "$DEV" ] && chgrp nut "$DEV" && chmod 660 "$DEV"'
```

Это компенсирует несрабатывающее udev-правило, оставляя сам драйвер непривилегированным (запускать драйвер от root — плохая практика безопасности, так как он парсит данные с USB-устройства).

## ⚠️ Важно: `pollonly` для стабильности USB HID

Дешёвые CyberPower часто криво реализуют USB interrupt-endpoint, из-за чего драйвер ловит:

```
nut_libusb_get_interrupt: Input/Output Error
Reconnecting...
```

**Лечение** — флаг `pollonly` в `ups.conf`. Он отключает асинхронные interrupt-уведомления и переводит драйвер на чистый polling (драйвер сам опрашивает ИБП). Это документированное решение для проблемных USB-ИБП, снижает частоту I/O-ошибок.

```
[myups]
    driver = usbhid-ups
    port = auto
    vendorid = 0764
    productid = 0501
    offdelay = 120
    ondelay = 240
    pollonly
    pollfreq = 30
    desc = "CyberPower UT2200EG"
```

## Как это работает

### Цепочка событий при отключении питания

**Кратковременное отключение (< 15 минут):**
UPS → ONBATT → upsmon запускает таймер 900 сек → свет вернулся → ONLINE → таймер отменён → все хосты работают дальше.

**Длительное отключение (> 15 минут):**
UPS → ONBATT → таймер 900 сек досиживает → primary объявляет FSD → все хосты (primary + secondary) синхронно запускают shutdown → primary после завершения своего shutdown командует killpower ИБП → ИБП ждёт `offdelay` и обесточивает выход.

**Разряд батареи до истечения таймера:**
UPS → ONBATT → ... → `LOWBATT` (при 10% заряда или прогнозе runtime < 5 мин) → немедленный FSD → shutdown без ожидания таймера.

**Потеря USB-связи с UPS:**
`COMMBAD` → только лог, shutdown НЕ выполняется (потеря связи ≠ потеря питания).

### Механизм killpower и offdelay

Ключевой момент координации между primary и secondary. Отсчёт `offdelay` начинается **не с момента FSD**, а с момента когда primary завершил свой shutdown и отправил команду killpower ИБП (это происходит из systemd-shutdown hook `nutshutdown` на самой последней стадии выключения primary).

Дедлайн для каждого secondary: он должен полностью завершить shutdown в пределах `(время shutdown primary) + offdelay`. Поскольку все хосты стартуют shutdown одновременно по FSD, а `offdelay = 120` секунд даёт большой запас, secondary успевают завершиться штатно.

Если `offdelay` слишком мал (заводское значение 20 сек), ИБП обесточивает выход раньше, чем secondary успевают выключиться — они получают жёсткое обесточивание посреди shutdown. Значение 120 сек подобрано с запасом под полный shutdown самого медленного secondary.

### Параметры таймеров

| Параметр | Значение | Файл | Что значит |
|---|---|---|---|
| Таймер выключения на батарее | 900 сек (15 мин) | `upssched.conf` | Сколько ждать после ONBATT до shutdown |
| offdelay | 120 сек | `ups.conf` | Задержка ИБП после killpower до обесточивания выхода |
| ondelay | 240 сек | `ups.conf` | Задержка ИБП перед повторным включением выхода (должна быть > offdelay) |
| HOSTSYNC | 90 сек | `upsmon.conf` | Timeout ожидания дисконнекта secondary-хостов от upsd |
| POLLFREQ | 5 сек | `upsmon.conf` | Как часто upsmon опрашивает upsd |
| pollinterval | 15 сек | `ups.conf` | Как часто драйвер опрашивает UPS |
| battery.charge.low | 10 % | прошивка ИБП | Порог LOWBATT по заряду |
| battery.runtime.low | 300 сек | прошивка ИБП | Порог LOWBATT по прогнозу автономки |

### Восстановление после отключения

Для автоматического включения хостов после возврата питания в BIOS материнки должна быть настройка **"Restore AC Power Loss" → Power On**. Значение **Last State** ненадёжно: если хост выключился сам через shutdown (а не от пропадания питания), при коротком power-cycle ИБП он может остаться выключенным. **Power On** гарантирует включение при любом появлении питания.

Учитывать взаимодействие с внешним прерывателем/реле, если он стоит перед ИБП: задержка возврата питания на прерывателе должна быть меньше NUT-таймера, иначе при коротком моргании хосты уйдут в shutdown до того как прерыватель вернёт питание.

## Troubleshooting

### Driver not connected / Data stale

```bash
# Проверить, все ли сервисы живы
systemctl status nut-driver@myups.service nut-server.service nut-monitor.service

# Проверить, видит ли система устройство
lsusb | grep -i '0764:0501'
```

Если устройство видно в `lsusb`, но драйвер падает с `insufficient permissions` — проблема в правах на USB (см. секцию про udev). Проверить права:

```bash
DEV=$(lsusb | grep -i '0764:0501' | sed -E 's|Bus ([0-9]+) Device ([0-9]+):.*|/dev/bus/usb/\1/\2|')
ls -la "$DEV"
# Должно быть root:nut. Если root:root — drop-in fixperms не отработал.
```

### insufficient permissions on everything

Устройство видно, но группа не `nut`. Дать права вручную и перезапустить драйвер:

```bash
systemctl stop nut-driver@myups.service
sleep 2
DEV=$(lsusb | grep -i '0764:0501' | sed -E 's|Bus ([0-9]+) Device ([0-9]+):.*|/dev/bus/usb/\1/\2|') \
  && chgrp nut "$DEV" && chmod 660 "$DEV"
systemctl start nut-driver@myups.service
```

Если проблема повторяется после каждой перезагрузки — проверить что drop-in `fixperms.conf` установлен и `systemctl daemon-reload` выполнен.

### Resource temporarily unavailable (11) — USB-устройство залипло

Симптом: `lsusb -d 0764:0501 -v` выдаёт `cannot read device status, Resource temporarily unavailable`. Устройство физически присутствует, но USB-контроллер ИБП завис.

Лечение по возрастанию радикальности:

```bash
# 1. Программный usbreset (помогает чаще всего)
systemctl stop nut-driver@myups.service
usbreset 0764:0501
sleep 3
# дать права и поднять драйвер (см. выше)

# 2. Если не помог — физически передёрнуть USB-кабель со стороны ИБП
```

### USB disconnect/reconnect циклом (нестабильный контакт)

Симптом в `dmesg`:

```
usb 1-5: USB disconnect, device number 14
usb 1-5: new low-speed USB device number 15 using xhci_hcd
usb 1-5: USB disconnect, device number 15
...
```

Устройство само отваливается и переподключается каждые несколько секунд, номер растёт. Это **физическая проблема контакта**, не лечится софтом. Причины по вероятности: неплотно воткнутый разъём (частая причина после перекоммутации), деградировавший USB-кабель, пережатый кабель, плохой USB-порт. Порядок действий: плотно передёрнуть оба конца кабеля, проверить что кабель не пережат, при необходимости заменить кабель на качественный USB-A → USB-B (до 1.5 м), попробовать другой USB-порт.

Проверка стабильности контакта (счётчик USB-событий не должен расти):

```bash
BEFORE=$(dmesg | grep -c 'usb 1-')
sleep 30
AFTER=$(dmesg | grep -c 'usb 1-')
[ "$BEFORE" = "$AFTER" ] && echo "СТАБИЛЬНО" || echo "ЕСТЬ реконнекты"
```

### Проверка подключённых клиентов

На primary можно посмотреть какие secondary сейчас подключены к upsd:

```bash
upsc -c myups@localhost
# Должны быть: 127.0.0.1 (primary), 192.168.10.11 (PVE-Mini), 192.168.10.15 (PBS)
```

### Debug-режим драйвера

```bash
# Остановить systemd-юнит драйвера перед ручным запуском
systemctl stop nut-driver@myups.service

# Запустить в foreground с debug-выводом
/usr/lib/nut/usbhid-ups -a myups -DD
```

### Изменение параметров ИБП на лету

Параметры `offdelay`/`ondelay` в `ups.conf` применяются при старте драйвера. Для чтения/записи на работающем ИБП нужен пользователь с `actions = SET` в `upsd.users`:

```bash
# Посмотреть доступные для записи параметры
upsrw myups@localhost

# Изменить (значение сбросится при рестарте драйвера, если не прописано в ups.conf)
upsrw -s ups.delay.shutdown=120 -u <admin_user> -p '<PASSWORD>' myups@localhost
```

Постоянные значения задаются только через `offdelay`/`ondelay` в `ups.conf`, так как ИБП не сохраняет их в энергонезависимой памяти.

## Тестирование shutdown

Полный тест разрушительный (все хосты реально выключатся). Временно снизить таймер для быстрого теста, потом вернуть:

```bash
# На primary: снизить таймер до 60 сек
sed -i 's|START-TIMER onbatt 900|START-TIMER onbatt 60|' /etc/nut/upssched.conf
systemctl restart nut-monitor

# Открыть логи на всех хостах:
#   journalctl -f -t nut-monitor -t upssched -t wall -u pve-guests
# Выдернуть кабель питания ИБП из розетки, наблюдать цепочку.
# После теста вернуть 900:
sed -i 's|START-TIMER onbatt 60|START-TIMER onbatt 900|' /etc/nut/upssched.conf
systemctl restart nut-monitor
```

Признак штатного завершения в логах прошлой загрузки каждого хоста:

```bash
journalctl -b -1 --no-pager | grep -iE 'all VMs and CTs|Journal stopped|poweroff.target'
```

Отсутствие этих строк (лог обрывается посреди остановки гостей) означает, что хост был обесточен до завершения shutdown — надо увеличивать `offdelay`.

## Связанные материалы

- [NUT documentation](https://networkupstools.org/docs/)
- [usbhid-ups man page](https://networkupstools.org/docs/man/usbhid-ups.html)
- [CyberPower UT2200EG](https://www.cyberpower.com/eu/en/product/sku/ut2200eg)
