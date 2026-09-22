# nut

Network UPS Tools: драйверы ИБП, `upsd` (сервер данных), `upsmon` (агент
выключения).

## Зачем, если есть роль `apcupsd`

apcupsd принимает решение о выключении **сам**, сравнивая пороги `MINUTES` и
`BATTERYLEVEL` с тем, что отдаёт ИБП. Если железка остаточное время не считает
и всегда возвращает ноль, условие «осталось меньше трёх минут» истинно
**всегда** — и хост гаснет через минуту после первого же моргания света при
полной батарее. На `gw.ventcomplex.ru` так и произошло 11.07.2026.

upsmon порогов не вычисляет: он гасит хост по флагу `LB`, который поднимает
сама железка. Для ИБП, чьи SNMP-карты остаток не сообщают, это единственная
корректная модель.

Подробности замеров — в памяти, `project-powerups`.

## Переменные

| | |
|---|---|
| `nut_enable` | гейт, по умолчанию `'false'` |
| `nut_mode` | `standalone` / `netserver` / `netclient` |
| `nut_ups` | список секций `ups.conf` |
| `nut_users` | пользователи `upsd`, пароли **только из vault** |
| `nut_upsd_listen` | адреса `upsd`, по умолчанию `127.0.0.1:3493` |
| `nut_upsmon_monitor` | что мониторить; **пусто = upsmon не поднимается** |
| `nut_upsmon_shutdowncmd` | по умолчанию `systemctl poweroff` |
| `nut_conflicting_services` | демоны за тот же ИБП — гасим и убираем из автозапуска |
| `nut_ups[].options` | пары `ключ: значение` → `ключ = значение` в секции; **пустое значение** (`''`) даёт опцию-флаг без `=`, как `battery_voltage_reports_one_pack` |
| `nut_serial_group` | группа serial-портов (`uucp` на Arch, `dialout` на Debian); при непустом значении пользователь `nut` добавляется в неё — иначе драйвер не откроет `/dev/ttyUSB*`. ИБП с USB-переходником внутри (CH341/PL2303) видны именно так |
| `nut_notify_telegram_enable` | мгновенные уведомления о событиях ИБП в Telegram через NOTIFYCMD upsmon; скрипт кладёт роль, токен/чат из vault (`nut_notify_telegram_token`, `nut_notify_telegram_chat_id`) |
| `nut_notify_telegram_messages` | текст по типу события (ONBATT, ONLINE, LOWBATT, REPLBATT, COMMBAD, COMMOK, NOCOMM, FSD, SHUTDOWN); чего нет в словаре — не шлётся |
| `nut_upsmon_shutdowncmd: '/bin/true'` | режим «только уведомления»: upsmon работает, события шлёт, хост при FSD не гасит |

### Секция ИБП

`options` уходит в `ups.conf` как есть — туда кладётся всё, что специфично
для драйвера:

```yaml
nut_ups:
  - name: 'ventusb'
    driver: 'nutdrv_qx'
    port: 'auto'
    desc: 'Powerman WPHVT2K0L'
    options:
      vendorid: '0665'
      productid: '5161'
      subdriver: 'cypress'
```

## Грабли

**`upsd.users` — строго `0640 root:nut`.** В файле пароли, а `upsd` читает его
уже сбросив права до пользователя `nut`. Другие права — либо отказ старта,
либо пароли наружу.

**`ups.conf` читает не драйвер, а `nut-driver-enumerator`** — он создаёт и
удаляет юниты `nut-driver@<имя>`. Поэтому при правке секций мало перезапустить
драйвер: сначала перечитывается список, иначе новая секция не появится.
Роль делает это сама через handler.

**Конфликтующий демон занимает устройство.** Если на хосте остался включённым
`apcupsd`, после ребута хозяином USB станет тот, кто стартовал первым, а второй
молча не поднимется. Отсюда `nut_conflicting_services`.

**ИБП, воткнутый до установки NUT, останется недоступным.** Правила udev из
пакета срабатывают на событие подключения, поэтому устройство остаётся
`root:root`, а драйвер падает:

```
libusb1: Could not open any HID devices: insufficient permissions on everything
```

Роль перечитывает правила и переигрывает события для USB сама, когда среди
драйверов есть USB-шный (`nut_usb_drivers`). Иначе лечится только
передёргиванием кабеля.

**Пустой `nut_upsmon_monitor` роль обеспечивает маской, а не пропуском.**
У `nut.target` статический `Wants=nut-monitor.service`, поэтому после
перезагрузки upsmon поднялся бы со штатным конфигом пакета — без единой строки
`MONITOR` — и упал. При непустом списке маска снимается.

**`MONITOR` без пароля не подключится молча** — хост останется без выключения
по низкому заряду. Роль проверяет это assert'ом.

## Поддержка

Archlinux и Debian. На Debian пакеты называются `nut-server` / `nut-client`.

## netdata

`nut_netdata_enable: true` кладёт `/etc/netdata/go.d/upsd.conf` с заданием на
каждую секцию из `nut_ups`. Роль `netdata` — сабмодуль k0ste, коллектора
`upsd` в ней нет, но чужие файлы в `go.d` она не удаляет, поэтому конфиг
приходит отсюда.

`nut_netdata_drop_collectors: ['apcupsd']` снимает конфиг коллектора, чей
демон на хосте больше не живёт. Без этого netdata продолжит опрашивать мёртвый
сокет, и мониторинг поднимет ложный «collector offline».

**Выключение интеграции убирает и свой конфиг.** Задача снятия намеренно НЕ
под гейтом `nut_netdata_enable`: иначе при выключении `go.d/upsd.conf`
оставался бы на месте и коллектор продолжал бы опрашивать — та же ложная
тревога, от которой уходили.

**Имена метрик меняются:** было `netdata_apcupsd_*`, стало `netdata_upsd_*`.
Правила на промке надо переводить вместе с хостом.

## Уведомления в Telegram

```yaml
nut_notify_telegram_enable: true
nut_notify_telegram_token: !vault |   # токен бота
  ...
nut_notify_telegram_chat_id: '-1001234567890'
```

Требует непустого `nut_upsmon_monitor` — события рождает upsmon. Скрипт
`/usr/local/bin/nut-notify-telegram` (0750 root:nut) получает `NOTIFYTYPE`,
`UPSNAME` и текст события, шлёт `<текст события из словаря>\nХост: … · ИБП: …`.
Если задать свои `nut_upsmon_notifycmd`/`nut_upsmon_notify`, они имеют
приоритет. Уведомления без выключения хоста: `nut_upsmon_shutdowncmd: '/bin/true'`.

## Сторож драйвера

`usbhid-ups` переживает не всякое переподключение USB. 22.09.2026 на lealav ИБП
отвалился и через секунду вернулся с новым номером устройства; драйвер остался
со старым хэндлом и три с половиной часа отвечал «No such device», а upsmon всё
это время слал «ИБП недоступен» каждые пять минут. Лечится перезапуском
`nut-driver@<ups>`.

`nut_driver_watchdog_enable: true` ставит таймер, который раз в
`nut_driver_watchdog_interval` дёргает `upsc <ups> ups.status` и после
`nut_driver_watchdog_failures` подряд неудач перезапускает драйвер. Пишет в
syslog под тегом `nut-driver-watchdog`.

## Шум от NOCOMM

`NOCOMM` в upsmon повторяется каждые `NOCOMMWARNTIME` секунд, пока связи нет;
у NUT это 300 по умолчанию. С Telegram получается сообщение раз в пять минут.
Поэтому в роли `NOCOMMWARNTIME` поднят до 1800, а сам `NOCOMM` по умолчанию
пишется только в syslog (`nut_notify_telegram_events_syslog_only`): о разрыве
уже сообщает `COMMBAD`, о восстановлении `COMMOK`.
