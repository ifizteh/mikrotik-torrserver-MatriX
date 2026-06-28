# TorrServer MatriX на MikroTik (RouterOS 7.23+)

Установка [TorrServer MatriX](https://github.com/YouROK/TorrServer) в контейнере MikroTik на современной прошивке RouterOS 7.23+.

> **Основано на работе [@Skaystk](https://github.com/Skaystk/mikrotik-torrserver-MatriX/tree/Mikrotik-torrserver).**
> Базовая инструкция и подготовка образа взяты из его репозитория — спасибо автору.
> Этот форк адаптирует процесс под RouterOS 7.21+, где изменился синтаксис монтирования (`mountlists` вместо `mounts`) и ряд команд `/container`.

---

## Что изменилось в RouterOS 7.21–7.23

В новых прошивках MikroTik переработал подсистему контейнеров. По сравнению с оригинальной инструкцией:

| Оригинал (старые версии) | RouterOS 7.21+ |
|---|---|
| `/container mount add name=... src=... dst=...` | `/container mounts add src=... dst=... list=...` |
| параметр `mounts=cfg,log,torr` при создании | параметр `mountlists=<имя_списка>` |
| маунт привязывался по имени | маунты группируются в **список** (`list=`) |

Ключевое: монтирования теперь объединяются в именованный список, и контейнер ссылается на этот список целиком через `mountlists=`.

---

## Требования

- MikroTik с поддержкой контейнеров (ARM64 — например hAP ax3) на RouterOS 7.23+
- Включённый режим контейнеров (`device-mode`)
- USB-накопитель или иной внешний диск, отформатированный в ext4
- Образ TorrServer под нужную архитектуру (см. ниже)

---

## 1. Подготовка образа на ПК

На macOS/Linux с установленным `skopeo` сохраняем multi-arch образ TorrServer в локальный TAR под архитектуру роутера (arm64 для hAP ax3):

```bash
brew install skopeo   # macOS; на Linux — пакетным менеджером дистрибутива

skopeo copy \
  --override-os linux \
  --override-arch arm64 \
  --override-variant v8 \
  docker://ghcr.io/yourok/torrserver:latest \
  docker-archive:torrserver-arm64.tar:yourok/torrserver:latest
```

Полученный `torrserver-arm64.tar` загрузите на роутер в корень USB (`usb1`).

---

## 2. Включение режима контейнеров

Если ещё не включён:

```routeros
/system device-mode update container=yes
# подтвердите по запросу (кнопкой reset или холодной перезагрузкой)
```

---

## 3. Подготовка USB и директорий

```routeros
/disk print
# при необходимости форматирование (СОТРЁТ ДАННЫЕ):
# /disk format usb1 filesystem=ext4
```

Создайте структуру папок для данных TorrServer:

```routeros
/file add name=usb1/opt/ts/config   type=directory
/file add name=usb1/opt/ts/log      type=directory
/file add name=usb1/opt/ts/torrents type=directory
/file add name=usb1/containers      type=directory
/file add name=usb1/container-tmp   type=directory
```

---

## 4. Сеть для контейнера (veth)

> Если у вас уже есть veth и bridge для контейнеров (например от mihomo/AdGuard), используйте их и пропустите этот шаг. В примере ниже — отдельный veth `veth-torrserve` с адресом `192.168.89.3/24` в существующем контейнерном bridge с gateway `192.168.89.1`.

```routeros
/interface veth add name=veth-torrserve address=192.168.89.3/24 gateway=192.168.89.1
/interface bridge port add bridge=container interface=veth-torrserve
```

> Замените `container` на имя вашего контейнерного моста, а адреса — на свою подсеть.

---

## 5. Реестр и tmpdir

```routeros
/container config set registry-url=https://registry-1.docker.io tmpdir=usb1/container-tmp
```

---

## 6. Переменные окружения

Группа переменных `tsM`, которую затем подключим к контейнеру:

```routeros
/container envs add list=tsM key=TS_CONF_PATH value=/opt/ts/config
/container envs add list=tsM key=TS_LOG_PATH  value=/opt/ts/log
/container envs add list=tsM key=TS_TORR_DIR  value=/opt/ts/torrents
/container envs add list=tsM key=TS_PORT      value=8090
```

---

## 7. Монтирования (новый синтаксис 7.21+)

Здесь главное отличие от старой инструкции — параметр `list=`, объединяющий маунты в группу:

```routeros
/container mounts add src=usb1/opt/ts/config   dst=/opt/ts/config   list=tsM
/container mounts add src=usb1/opt/ts/log      dst=/opt/ts/log      list=tsM
/container mounts add src=usb1/opt/ts/torrents dst=/opt/ts/torrents list=tsM
```

Проверка:

```routeros
/container mounts print
```

---

## 8. Создание контейнера

Контейнер ссылается на список монтирований через `mountlists=` и на переменные через `envlists=`:

```routeros
/container add name=TorrServer-MatriX \
    file=usb1/torrserver-arm64.tar \
    interface=veth-torrserve \
    root-dir=usb1/containers/ts-MatriX \
    mountlists=tsM \
    envlists=tsM \
    start-on-boot=yes
```

Дождитесь распаковки образа (статус сменится с `extracting` на `stopped`):

```routeros
/container print
```

---

## 9. Запуск

```routeros
/container start [find name=TorrServer-MatriX]
```

Убедитесь, что процесс поднялся внутри контейнера:

```routeros
/container shell [find name=TorrServer-MatriX]
# внутри:
ps aux
wget -O- http://localhost:8090
exit
```

---

## 10. Проброс порта на веб-интерфейс

Чтобы открыть интерфейс TorrServer с устройств вашей LAN через адрес роутера:

```routeros
/ip firewall nat add chain=dstnat comment=TorrServer \
    protocol=tcp dst-address=192.168.3.1 dst-port=8090 \
    to-addresses=192.168.89.3 to-ports=8090
```

`dst-address=<IP роутера>` ограничивает правило обращениями именно к адресу роутера (обычно это и нужно — заходим на `http://<IP_роутера>:8090`). При желании этот параметр можно опустить, тогда dst-nat сработает для порта 8090 на любом адресе назначения:

```routeros
/ip firewall nat add chain=dstnat comment=TorrServer \
    protocol=tcp dst-port=8090 \
    to-addresses=192.168.89.3 to-ports=8090
```

> Если после добавления правила интерфейс сразу не открылся, сбросьте conntrack: `/ip firewall connection remove [find dst-address~"8090"]` и повторите.

Откройте в браузере:

```
http://<IP_роутера>:8090
```

Например `http://192.168.3.1:8090`.

---

## Отладка

**Контейнер `running`, но интерфейс не открывается:**

1. Проверьте, что TorrServer слушает порт изнутри:
   ```routeros
   /container shell [find name=TorrServer-MatriX]
   ps aux                       # должен быть процесс torrserver ... --port 8090
   wget -O- http://localhost:8090
   exit
   ```
2. Проверьте доступность контейнера с роутера:
   ```routeros
   /ping 192.168.89.3 count=3
   /tool fetch url="http://192.168.89.3:8090" mode=http output=user as-value
   ```
3. Проверьте, что маршрут до контейнерной сети активен:
   ```routeros
   /ip route print where dst-address~"192.168.89"
   ```
4. Проверьте NAT-правило (не disabled, корректные `to-addresses`/`to-ports`):
   ```routeros
   /ip firewall nat print where comment=TorrServer
   ```
   Если правило выглядит верно, но не срабатывает — сбросьте conntrack:
   ```routeros
   /ip firewall connection remove [find dst-address~"8090"]
   ```

**Логи контейнера:**

```routeros
/container set [find name=TorrServer-MatriX] logging=yes
/container stop  [find name=TorrServer-MatriX]
/container start [find name=TorrServer-MatriX]
/log print where topics~"container"
```

---

## Благодарности

- [@Skaystk](https://github.com/Skaystk/mikrotik-torrserver-MatriX) — оригинальная инструкция и подготовка образа.
- [YouROK/TorrServer](https://github.com/YouROK/TorrServer) — собственно TorrServer.
