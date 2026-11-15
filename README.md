# Netis NX32U V1 — установка OpenWrt 25.12.6

Пошаговая инструкция по переходу с заводской прошивки Netis NX32U V1 на OpenWrt 25.12.6.

> **Важно:** инструкция составлена по фактически выполненной установке. Не применяйте её к другим ревизиям роутера без проверки совместимости образов и разметки Flash.

## Итоговое состояние

| Компонент | Состояние после установки |
|---|---|
| BootROM | Заводской |
| BL2 | Заводской |
| FIP | OpenWrt U-Boot |
| UBI | Подготовлен для OpenWrt |
| OpenWrt | 25.12.6 |
| RootFS | SquashFS |

**BL2 не обновлялся.** Файл `preloader.bin` был скачан, но не использовался. Заводской BL2 успешно запускает записанный FIP.

## Что понадобится

- Netis NX32U V1.
- ПК с Windows и Ethernet-портом.
- SSH-клиент (доступен в Windows через PowerShell).
- [Tftpd64](https://github.com/PJO2/tftpd64/releases/).
- Четыре файла OpenWrt из [официального каталога релиза 25.12.6](https://downloads.openwrt.org/releases/25.12.6/targets/mediatek/filogic/).

Файлы прошивки:

```text
openwrt-25.12.6-mediatek-filogic-netis_nx32u-bl31-uboot.fip
openwrt-25.12.6-mediatek-filogic-netis_nx32u-preloader.bin
openwrt-mediatek-filogic-netis_nx32u-initramfs-recovery.itb
openwrt-25.12.6-mediatek-filogic-netis_nx32u-squashfs-sysupgrade.itb
```

## Последовательность работ

| Шаг | Что сделали | Зачем |
|---:|---|---|
| 0 | Подключились к заводской прошивке по LAN и SSH | Подготовили роутер к работе |
| 1 | Скачали четыре файла OpenWrt | Получили загрузчик, Recovery и постоянную прошивку |
| 2 | Проверили SHA256 | Проверили целостность файлов |
| 3 | Сохранили дампы MTD на ПК | Создали резервные копии заводской Flash |
| 4 | Настроили TFTP на ПК | Подготовили передачу Recovery роутеру |
| 5 | Записали FIP | Установили загрузчик OpenWrt |
| 6 | Проверили FIP командой `mtd verify` | Убедились в корректности записи |
| 7 | Загрузили Recovery через TFTP | Запустили временную среду OpenWrt |
| 8 | Скопировали sysupgrade-образ | Подготовили постоянную прошивку |
| 9 | Подготовили UBI | Подготовили NAND-раздел |
| 10 | Выполнили `sysupgrade -n` | Установили OpenWrt во Flash без старых настроек |

---

## 0. Подключение к заводской прошивке

Подключили ПК кабелем к LAN1 роутера.

**Сетевые настройки Ethernet на ПК:**

| Параметр | Значение |
|---|---|
| IP-адрес | `192.168.1.10` |
| Маска | `255.255.255.0` |
| Шлюз | `192.168.1.1` |

Подключились к роутеру по SSH из PowerShell:

```powershell
# Подключение к заводской прошивке
ssh root@192.168.1.1
```

Использовали пароль администратора заводского веб-интерфейса.

## 1. Загрузка файлов OpenWrt

Скачали четыре файла из [официального каталога OpenWrt 25.12.6](https://downloads.openwrt.org/releases/25.12.6/targets/mediatek/filogic/).

Сохранили их в каталог:

```text
C:\NX32U-OpenWrt
```

## 2. Проверка SHA256

**PowerShell на ПК:**

```powershell
# Выводим SHA256 всех скачанных файлов
Get-ChildItem "C:\NX32U-OpenWrt" -File | Get-FileHash -Algorithm SHA256
```

Сравнили полученные значения с контрольными суммами из официального файла `sha256sums` в каталоге релиза.

## 3. Резервное копирование MTD

До изменения загрузчика сохранили полный дамп Flash и отдельные разделы заводской прошивки.

**PowerShell на ПК:**

```powershell
# Создаём каталог для резервных копий
New-Item -ItemType Directory -Force "C:\NX32U-backup"

# Сохраняем полный дамп Flash
cmd.exe /d /c 'ssh root@192.168.1.1 "cat /dev/mtd0" > "C:\NX32U-backup\mtd0-full.bin"'

# Сохраняем отдельные разделы
cmd.exe /d /c 'ssh root@192.168.1.1 "cat /dev/mtd1" > "C:\NX32U-backup\mtd1_BL2.bin"'
cmd.exe /d /c 'ssh root@192.168.1.1 "cat /dev/mtd2" > "C:\NX32U-backup\mtd2_u-boot-env.bin"'
cmd.exe /d /c 'ssh root@192.168.1.1 "cat /dev/mtd3" > "C:\NX32U-backup\mtd3_Factory.bin"'
cmd.exe /d /c 'ssh root@192.168.1.1 "cat /dev/mtd4" > "C:\NX32U-backup\mtd4_FIP.bin"'
cmd.exe /d /c 'ssh root@192.168.1.1 "cat /dev/mtd5" > "C:\NX32U-backup\mtd5_ubi.bin"'
```

Резервные копии сохранили на ПК в `C:\NX32U-backup`.

> **Не публикуйте дампы MTD в открытом репозитории.** Они могут содержать индивидуальные данные устройства, заводские параметры и калибровки Wi-Fi.

## 4. Настройка TFTP на ПК

Создали каталог:

```text
C:\NX32U-TFTP
```

Запустили [Tftpd64](https://github.com/PJO2/tftpd64/releases/) и настроили:

| Параметр | Значение |
|---|---|
| Current Directory | `C:\NX32U-TFTP` |
| Server interface | `192.168.1.254` |
| TFTP Server | Включён |

На Ethernet-адаптере ПК для этого этапа установили IP-адрес `192.168.1.254/24`.

Поместили в каталог TFTP файл Recovery:

```text
openwrt-mediatek-filogic-netis_nx32u-initramfs-recovery.itb
```

Имя файла оставили без изменений.

## 5. Запись FIP

**PowerShell на ПК:**

```powershell
# Копируем FIP во временный каталог роутера
scp -O "C:\NX32U-OpenWrt\openwrt-25.12.6-mediatek-filogic-netis_nx32u-bl31-uboot.fip" root@192.168.1.1:/tmp/
```

Параметр `-O` включает старый протокол SCP, поскольку на заводской прошивке отсутствовал совместимый SFTP subsystem.

**SSH роутера:**

```sh
# Записываем новый FIP в раздел FIP
mtd write /tmp/openwrt-25.12.6-mediatek-filogic-netis_nx32u-bl31-uboot.fip FIP

# Сохраняем изменения во Flash
sync
```

## 6. Проверка FIP

**SSH роутера:**

```sh
# Сравниваем записанный раздел с исходным файлом
mtd verify /tmp/openwrt-25.12.6-mediatek-filogic-netis_nx32u-bl31-uboot.fip FIP
```

Проверка завершилась успешно: содержимое раздела совпало с образом.

## 7. Загрузка Recovery через TFTP

1. Запустили Tftpd64 и открыли вкладку **TFTP Server**.
2. Выбрали сетевой интерфейс `192.168.1.254`.
3. Выключили питание роутера.
4. Зажали кнопку **Reset** и включили питание.
5. Удерживали Reset примерно 10 секунд, затем отпустили.
6. Загрузчик запросил Recovery по TFTP. Передачу файла подтвердили по журналу Tftpd64.

После загрузки роутер запустился во временной среде OpenWrt Recovery.

## 8. Копирование постоянной прошивки

**PowerShell на ПК:**

```powershell
# Передаём sysupgrade-образ во временный каталог роутера
scp -O "C:\NX32U-OpenWrt\openwrt-25.12.6-mediatek-filogic-netis_nx32u-squashfs-sysupgrade.itb" root@192.168.1.1:/tmp/
```

## 9. Подготовка UBI

**SSH роутера (Recovery):**

> **Внимание:** форматирование удаляет текущее содержимое раздела UBI. Команды выполнялись на `/dev/mtd4` роутера NX32U в Recovery.

```sh
# Отключаем UBI от NAND-раздела
ubidetach -p /dev/mtd4

# Форматируем NAND-раздел UBI
ubiformat -y /dev/mtd4

# Подключаем UBI обратно
ubiattach -p /dev/mtd4

# Создаём служебные тома загрузочной среды
ubimkvol /dev/ubi0 -N ubootenv -s 1MiB
ubimkvol /dev/ubi0 -N ubootenv2 -s 1MiB

# Проверяем состояние UBI и созданные тома
ubinfo -a
```

Команды завершились успешно. Были созданы тома `ubootenv` и `ubootenv2`.

## 10. Установка OpenWrt

**SSH роутера (Recovery):**

```sh
# Проверяем совместимость образа перед установкой
sysupgrade -T /tmp/openwrt-25.12.6-mediatek-filogic-netis_nx32u-squashfs-sysupgrade.itb

# Устанавливаем OpenWrt без сохранения старых настроек
sysupgrade -n /tmp/openwrt-25.12.6-mediatek-filogic-netis_nx32u-squashfs-sysupgrade.itb
```

После завершения установки роутер перезагрузился с постоянной прошивкой OpenWrt.

---

## Проверка результата

После перезагрузки проверили модель и версию системы:

```sh
# Информация о модели, версии OpenWrt и типе RootFS
ubus call system board

# Таблица MTD-разделов
cat /proc/mtd
```

Подтверждённый результат:

- Модель: `netis NX32U`
- Board name: `netis,nx32u`
- Версия: `25.12.6`
- RootFS: `squashfs`
- BL2 остался заводским.
- FIP заменён на OpenWrt и проверен через `mtd verify`.

## Примечание

Инструкция описывает конкретную конфигурацию Netis NX32U V1 и OpenWrt 25.12.6. Перед повторением процедуры проверьте модель и ревизию устройства, актуальность файлов прошивки и соответствие MTD-разметки.
