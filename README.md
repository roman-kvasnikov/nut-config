# NUT config for Proxmox VE

Конфигурация [Network UPS Tools](https://networkupstools.org/) для PVE-хоста с подключённым по USB ИБП на базе чипа Cypress (INNO TECH, USB ID `0665:5161` — типичный OEM-клон, встречается в Powercom, Ippon, SVEN и других бытовых линейках). Сетап standalone: один хост, один UPS, автоматический корректный shutdown при длительном отключении питания.

## Что внутри

| Файл | Назначение |
|---|---|
| `nut.conf` | Режим работы NUT (`netserver`) |
| `ups.conf` | Описание UPS-устройства для драйвера |
| `upsd.conf` | Настройки сетевого сервера `upsd` |
| `upsd.users.example` | Шаблон пользователей с плейсхолдерами |
| `upsmon.conf.example` | Шаблон монитора с плейсхолдером пароля |
| `upssched.conf` | Планировщик событий (таймеры) |
| `upssched-cmd` | Скрипт-обработчик для upssched |

Реальные `upsd.users` и `upsmon.conf` в репо не хранятся — они в `.gitignore`, так как содержат пароли NUT-пользователей.

## На чём тестировалось

- Proxmox VE 9 (Debian 13 trixie)
- NUT 2.8.1
- ИБП с чипом Cypress Semiconductor USB-to-Serial (`0665:5161`), INNO TECH
- Протокол связи: Megatec Q1 (определяется драйвером автоматически)

## Установка

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
PEANUTPASS=$(openssl rand -base64 18 | tr -d '/+=')
sed -i "s/CHANGE_ME_MONUSER_PASSWORD/$MONPASS/g" /etc/nut/upsd.users /etc/nut/upsmon.conf
sed -i "s/CHANGE_ME_PEANUT_PASSWORD/$PEANUTPASS/g" /etc/nut/upsd.users

# 4. Включить и запустить
systemctl enable --now nut-driver.target nut-server.service nut-monitor.service

# 5. Проверить
upsc myups@localhost
```

## ⚠️ Важно: `langid_fix = 0x409` для Cypress UPS

Чипы Cypress CY7C63001 (и клоны) в USB-контроллерах ИБП часто возвращают сломанный или пустой USB LangID descriptor. libusb не может прочитать строки iManufacturer/iProduct, и драйвер `nutdrv_qx` отваливается с:

```
[D1] nut_libusb_open get iManufacturer failed, retrying...
Device not supported!
```

**Лечение** — параметр `langid_fix = 0x409` в `ups.conf`. Он заставляет драйвер пропустить сломанное language negotiation и читать строки напрямую с фиксированным LangID `0x0409` (English US).

Это описано в [документации NUT](https://networkupstools.org/docs/man/nutdrv_qx.html), но в русскоязычном интернете информации мало — поэтому если ты нашёл этот репо через поиск «nutdrv_qx device not supported Cypress» или «INNO TECH UPS Linux», добавь `langid_fix = 0x409` в свою секцию в `ups.conf` и перезапусти `nut-driver@*.service`.

## Как это работает

### Сценарии при отключении питания

**Кратковременное отключение (< 10 минут):**
UPS → ONBATT → upsmon запускает таймер 600 сек → свет вернулся → ONLINE → таймер отменён → сервер работает дальше.

**Длительное отключение (> 10 минут):**
UPS → ONBATT → таймер 600 сек досиживает → `upsmon -c fsd` → `systemctl poweroff` → PVE гасит гостей и выключается.

**Разряд батареи до истечения таймера:**
UPS → ONBATT → ... → `LOWBATT` → немедленный `upsmon -c fsd` → shutdown.

**Потеря USB-связи с UPS:**
`COMMBAD` → только лог, shutdown НЕ выполняется (потеря связи ≠ потеря питания).

### Параметры таймеров

| Параметр | Значение | Файл | Что значит |
|---|---|---|---|
| Таймер выключения на батарее | 600 сек (10 мин) | `upssched.conf` | Сколько ждать после ONBATT до shutdown |
| POLLFREQ | 5 сек | `upsmon.conf` | Как часто upsmon опрашивает upsd |
| pollinterval | 3 сек | `ups.conf` | Как часто драйвер опрашивает UPS |
| NOCOMMWARNTIME | 600 сек | `upsmon.conf` | После скольких секунд без связи слать NOCOMM-алерты |
| HOSTSYNC | 15 сек | `upsmon.conf` | Timeout ожидания secondary-хостов |

### Восстановление после длительного отключения

Для автоматического включения сервера после возврата питания в BIOS материнки должна быть настройка **"Restore AC Power Loss" → Power On** (или **Last State**). Без этого сервер останется выключенным до ручного включения.

## Troubleshooting

### Driver not connected

```bash
# Проверить, все ли сервисы живы
systemctl status nut-driver@myups.service nut-server.service nut-monitor.service

# Типичная причина — stale PID-файл после падения драйвера
ls /run/nut/
systemctl stop nut-monitor.service nut-server.service
systemctl stop 'nut-driver@*.service'
pkill -9 -f nutdrv_qx
rm -f /run/nut/*.pid /run/nut/*.lock

systemctl start nut-driver.target
systemctl start nut-server.service
systemctl start nut-monitor.service
```

### Device not supported на Cypress-чипе

См. секцию про `langid_fix` выше.

### USB-устройство залипло (исчезает из `lsusb`)

После серии рестартов драйвера Cypress-чип может залипнуть настолько, что программный `usbreset` не помогает. Лечение — физический реплаг USB-кабеля между сервером и ИБП.

### Debug-режим драйвера

```bash
# Остановить systemd-юнит драйвера перед ручным запуском
systemctl stop nut-driver@myups.service

# Запустить в foreground с debug-выводом
runuser -u nut -- /lib/nut/nutdrv_qx -a myups -D -d 1
```

## Связанные материалы

- [NUT documentation](https://networkupstools.org/docs/)
- [nutdrv_qx man page](https://networkupstools.org/docs/man/nutdrv_qx.html)
